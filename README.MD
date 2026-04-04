# Home SOC Operations Lab
### SOC Analyst L1 — Threat Detection, Network Visibility & Incident Response

## Overview
A fully operational Security Operations Center deployed on a live Windows 11 
host using WSL2, simulating real-world SOC L1 analyst workflow including alert 
monitoring, network traffic analysis, threat triage, IOC investigation, and 
incident documentation.

## Architecture

| Component | Role | Address |
|---|---|---|
| Ubuntu 22.04 (WSL2) | Wazuh Manager + SIEM + Zeek Analysis | 172.23.201.127 |
| Windows 11 (Host) | Wazuh Agent + Sysmon + Monitored Endpoint | 172.23.192.1 |

> WSL2 IP statically assigned via `/etc/wsl.conf` boot command — persistent across reboots.

## Stack

| Tool | Version | Purpose | Status |
|---|---|---|---|
| Wazuh | 4.14.4 | SIEM + EDR + XDR | ✅ Active |
| Logstash | 7.17.29 | Alert + Zeek pipeline to OpenSearch | ✅ Active |
| Zeek | 8.0.5 | Network traffic analysis (pcap mode) | ✅ Active |
| Sysmon | 15.20 | Deep Windows process telemetry | ✅ Active |
| Wireshark | 4.6.4 | Live packet capture → pcap export | ✅ Active |
| TheHive | 5.2.16 | Incident case management (on-demand) | ✅ Operational |
| VirusTotal | — | IOC enrichment | ✅ Active (manual) |
| AbuseIPDB | — | IP reputation lookup | ✅ Active (manual) |
| OTX AlienVault | — | Threat intelligence feeds | ✅ Active (manual) |
| CyberChef | — | Log/IOC decoding | ✅ Active (manual) |

## OSI Coverage

| OSI Layer | Tool |
|---|---|
| Layer 7 — Application | Wazuh (app logs), Zeek (HTTP/DNS), Sysmon (process behavior) |
| Layer 4 — Transport | Zeek (conn.log — TCP/UDP sessions) |
| Layer 3 — Network | Zeek (IP traffic), Wazuh (firewall rules) |
| Endpoint (all layers) | Wazuh Agent + Sysmon (Windows) |

## Daily Workflow

- Capture live traffic via Wireshark → analyze with Zeek
- Monitor Wazuh alerts across active agents
- Triage Zeek `weird.log`, `conn.log`, `dns.log` for anomalies
- Investigate IOCs using VirusTotal, AbuseIPDB, whois
- Document incidents in TheHive (started on-demand)
- Update GitHub with findings and incident notes


## Incidents

| Case | Title | Severity | Verdict |
|---|---|---|---|
| 001 | active_connection_reuse — PLDT DNS Server (IPv6) | Low | Benign — ISP DNS resolver |

## Status
Active — updated regularly
Last updated: April 4, 2026

``` Daily Notes
4/4/2026
tralalero tralala
```

``` Important Notes
- TheHive runs **on-demand only** to preserve RAM (`docker-compose up/down`)
- Zeek operates in **pcap replay mode** — traffic captured on Windows via Wireshark, analyzed in WSL2
- WSL2 dynamically shares 8GB RAM with Windows host
```
