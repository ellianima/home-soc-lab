# SOC Lab Architecture

## Network Overview

| Component           | OS               | IP Address                                   | Role                               |
| ------------------- | ---------------- | -------------------------------------------- | ---------------------------------- |
| Ubuntu 22.04 (WSL2) | Ubuntu 22.04 LTS | 172.23.201.127 (static — persistent across reboots)) | SOC Manager                        |
| Windows 11          | Windows 11       | 172.23.192.1                                 | Monitored Endpoint (Windows Agent) |
| Kali Linux VM       | Kali Rolling     | TBD — 4GB RAM                                | Attacker 🔜 Planned                |
| Home Router         | ZTE F670L        | 192.168.1.1                                  | Gateway / DHCP Server              |

**Hypervisor:** Oracle VirtualBox (Host: Windows 11)
**SIEM Host:** WSL2 Ubuntu 22.04 (shares RAM dynamically with Windows)

> ✅ WSL2 IP is statically assigned via `/etc/wsl.conf` boot command. Persistent across reboots.

---

## Component Roles

### Ubuntu 22.04 WSL2 — SOC Manager (`172.23.201.127`)

The central nerve of the lab. All security tooling lives here inside WSL2.

| Tool            | Version                    | Purpose                              | Status     |
| --------------- | -------------------------- | ------------------------------------ | ---------- |
| Wazuh Manager   | 4.14.4                     | SIEM — collects, correlates, alerts  | ✅ Active  |
| Wazuh Indexer   | 4.14.4 (OpenSearch 2.19.4) | Log storage and indexing backend     | ✅ Active  |
| Wazuh Dashboard | 4.14.4                     | Alert visualization and dashboards   | ✅ Active  |
| Logstash        | 7.17.29                    | Alert pipeline (Wazuh → OpenSearch)  | ✅ Active  |
| Zeek            | 8.0.5                      | Network traffic capture and analysis | ✅ Active |
| TheHive         | 5.2 (Docker)               | Incident response case management    | 🔜 Planned |

**Key ports:**

- `1514/tcp` — Wazuh agent log ingestion
- `1515/tcp` — Wazuh agent enrollment
- `9200/tcp` — OpenSearch (Wazuh Indexer) API
- `443/tcp` — Wazuh Dashboard (HTTPS)
- `9000/tcp` — TheHive web interface (when deployed)
- `5044/tcp` — Logstash pipeline input (internal)

### Windows 11 — Monitored Endpoint (`172.23.192.1`)

**Hostname:** `Candy-Cat-Silly-Fun`
**Status:** ACTIVE

Physical host machine also acting as a monitored endpoint. Wazuh agent installed. Used for:

- Windows Event Log monitoring (4624 logon, 4625 failed logon, 4688 process creation)
- File Integrity Monitoring (FIM) — `C:\Users\Public`, `C:\Windows\System32`, `C:\ProgramData`
- Security Configuration Assessment (CIS Windows 11 Benchmark)
- Vulnerability Detection
- Future attack target for red team simulation

### Kali Linux VM — Attacker 🔜 Planned

**RAM:** 4GB allocated
**Network:** Bridged (same LAN as Windows host)
**Rules:** Zero SOC tools — pure attacker machine
Used for:

- Network reconnaissance (Nmap, Netdiscover)
- Exploitation (Metasploit)
- Credential attacks (Hydra, Hashcat)
- Web attacks (Burp Suite, SQLmap)
- Post-exploitation simulation

---

## Data Flow
```
[Windows Agent] ──TCP 1514──► [Wazuh Manager : WSL2 172.23.201.127]
│
├── alerts.json (local file)
│ │
│ └── Logstash reads ──► OpenSearch (port 9200)
│
├── Wazuh Dashboard queries OpenSearch
└── Cases opened → TheHive (planned)
[WSL2 eth0] ──passive sniff──► [Zeek] ──► conn.log / dns.log / http.log (planned)
[Kali VM] ──attacks──► [Windows 11] ──events──► [Wazuh Agent] ──► Manager (planned)
```
---

## Tool Interaction Map

```
Wazuh Manager
├── receives agent events (Windows logs, FIM, SCA, vulnerability data)
├── correlates against built-in + custom rules
├── writes alerts to /var/ossec/logs/alerts/alerts.json
└── (planned) forwards alerts to TheHive via API
Logstash
├── reads /var/ossec/logs/alerts/alerts.json
├── ships to OpenSearch index wazuh-alerts-4.x-YYYY.MM.DD
└── uses logstash-output-opensearch plugin (Filebeat incompatible with WSL2)
Wazuh Dashboard
├── queries OpenSearch for alert data
├── renders MITRE ATT&CK, FIM, Compliance, Vulnerability panels
└── accessible at https://172.23.201.127 or https://localhost
Zeek (planned)
├── listens on network interface in promiscuous mode
├── generates structured logs: conn.log, dns.log, http.log, notice.log
└── logs stored at /opt/zeek/logs/current/
TheHive (planned)
├── receives incident cases (manual or automated)
├── tracks observables (IPs, hashes, domains)
└── documents analyst actions and resolution
```

---

## OSI Layer Coverage

| OSI Layer              | Tool Monitoring It                                    | Status                         |
| ---------------------- | ----------------------------------------------------- | ------------------------------ |
| Layer 7 — Application  | Wazuh (Windows Event Logs, app logs), Zeek (HTTP/DNS) | ✅ Wazuh Active |
| Layer 6 — Presentation | Wazuh (SSL/TLS inspection via rules)                  | 🔜 Partial                     |
| Layer 4 — Transport    | Zeek (conn.log — TCP/UDP sessions)                    | ✅ Active                      |
| Layer 3 — Network      | Zeek (IP traffic), Wazuh (firewall rules)             | ✅ Zeek Active / 🔜 Wazuh firewall rules planned |
| Layer 2 — Data Link    | Zeek (ARP), Suricata (future)                         | ✅ Zeek Active / 🔜 Suricata planned  |
| Endpoint (all layers)  | Wazuh agent (Windows 11 — FIM, SCA, vuln detection)   | ✅ Active                      |

---

## Why WSL2 Instead of a Dedicated VM?

| Factor                 | VM-based Wazuh      | WSL2 Wazuh (current)              |
| ---------------------- | ------------------- | --------------------------------- |
| RAM cost               | Fixed 4–6GB         | Dynamic — shares with Windows     |
| Boot time              | ~2 min              | ~10 seconds                       |
| Dashboard access       | VM IP only          | localhost or WSL2 IP              |
| File access            | Separate filesystem | `/mnt/c/` bridges Windows         |
| Attack lab coexistence | Hard on 12GB RAM    | Comfortable — leaves 4GB for Kali |

---

## Known Limitations

| Limitation                      | Impact                             | Workaround                                  |
| ------------------------------- | ---------------------------------- | ------------------------------------------- |
| WSL2 IP changes on reboot       | FIXED                              | Static IP assigned via /etc/wsl.conf boot command|
| Filebeat incompatible with WSL2 | Cannot use official Wazuh pipeline | Replaced with Logstash + OpenSearch plugin  |
| Single Windows agent            | No Linux endpoint monitoring       | Kali VM agent planned for Phase 2           |
| No network tap                  | Zeek sees WSL2 traffic only        | Full LAN coverage when Zeek + Kali deployed |

---

_Cerveaux Labs — Home SOC Operations_
_github.com/ellianima/home-soc-lab_
