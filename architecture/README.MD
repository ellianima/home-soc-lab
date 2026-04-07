# SOC Lab Architecture

## Network Overview

| Component | OS | IP Address | Role |
|---|---|---|---|
| Ubuntu 22.04 (WSL2) | Ubuntu 22.04 LTS | 172.23.201.127 (static — persistent across reboots) | SOC Manager |
| Windows 11 | Windows 11 | 172.23.192.1 | Monitored Endpoint + Packet Capture |
| Kali Linux VM | Kali Rolling | 192.168.1.10 | Attacker |
| Home Router | ZTE F670L | 192.168.1.1 | Gateway / DHCP Server |

**Hypervisor:** Oracle VirtualBox (Host: Windows 11)
**SIEM Host:** WSL2 Ubuntu 22.04 (shares RAM dynamically with Windows)

> ✅ WSL2 IP statically assigned via `/etc/wsl.conf` boot command. Persistent across reboots.

---

## Component Roles

### Ubuntu 22.04 WSL2 — SOC Manager (`172.23.201.127`)

| Tool | Version | Purpose | Status |
|---|---|---|---|
| Wazuh Manager | 4.14.4 | SIEM — collects, correlates, alerts | ✅ Active |
| Wazuh Indexer | 4.14.4 (OpenSearch 2.19.4) | Log storage and indexing backend | ✅ Active |
| Wazuh Dashboard | 4.14.4 | Alert visualization and dashboards | ✅ Active |
| Logstash | 7.17.29 | Pipeline (Wazuh alerts + Zeek logs → OpenSearch) | ✅ Active |
| Zeek | 8.0.5 | Network traffic analysis (pcap replay mode) | ✅ Active |
| TheHive | 5.2.16 (Docker) | Incident response case management (on-demand) | ✅ Operational |
| Docker | 28.2.2 | Container runtime for TheHive stack | ✅ Active |

**Key ports:**

- `1514/tcp` — Wazuh agent log ingestion
- `1515/tcp` — Wazuh agent enrollment
- `9200/tcp` — OpenSearch (Wazuh Indexer) API
- `443/tcp` — Wazuh Dashboard (HTTPS)
- `9000/tcp` — TheHive web interface (on-demand)
- `5044/tcp` — Logstash pipeline input (internal)

---

### Windows 11 — Monitored Endpoint + Capture Station (`172.23.192.1`)

**Hostname:** `Candy-Cat-Silly-Fun`
**Status:** ACTIVE

| Tool | Version | Purpose | Status |
|---|---|---|---|
| Wazuh Agent | 4.14.4 | Ships Windows events to Wazuh Manager | ✅ Active |
| Sysmon | 15.20 | Deep process + network telemetry | ✅ Active |
| Wireshark | 4.6.4 | Live packet capture → .pcap export | ✅ Active |

**Sysmon config:** SwiftOnSecurity (`sysmonconfig-export.xml`)
**Wireshark capture interface:** Wi-Fi (physical NIC — real LAN traffic)
**pcap storage:** `C:\Users\elli\Desktop\JOB\PHASE A\SOC ANALYST L1\pcap folder\`

Windows monitoring coverage:
- Windows Event Logs (4624 logon, 4625 failed logon, 4688 process creation)
- Sysmon Event ID 1 (process creation), 3 (network connection), 11 (file creation)
- File Integrity Monitoring (FIM) — `C:\Users\Public`, `C:\Windows\System32`, `C:\ProgramData`
- Security Configuration Assessment (CIS Windows 11 Benchmark)
- Vulnerability Detection

---

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
[Windows 11]
├── Wazuh Agent ──TCP 1514──► [Wazuh Manager : WSL2 172.23.201.127]
│                                      │
│                               alerts.json
│                                      │
│                              Logstash reads ──► OpenSearch (port 9200)
│                                                        │
│                                              Wazuh Dashboard queries
│
├── Sysmon ──► Windows Event Log ──► Wazuh Agent ──► Manager
│
└── Wireshark captures Wi-Fi NIC ──► .pcap file
│
copied to WSL2 ~/zeek-analysis/
│
Zeek analyzes
│
conn.log / dns.log / http.log / ssl.log / weird.log
│
Logstash reads (JSON) ──► OpenSearch zeek-* index
│
Wazuh Dashboard (zeek-* pattern)
[TheHive — on demand]
├── docker-compose up → Cassandra + TheHive
├── Analyst opens cases for True Positives
├── Observables: IPs, domains, hashes, processes
└── docker-compose down → stops when investigation complete
```
---

## Tool Interaction Map
```
Wazuh Manager
├── receives agent events (Windows logs, Sysmon, FIM, SCA, vulnerability)
├── correlates against built-in + custom rules
├── writes alerts → /var/ossec/logs/alerts/alerts.json
└── forwards alerts to TheHive (planned — integration module)
Sysmon (Windows)
├── monitors process creation, network connections, file creation, registry
├── writes to Windows Event Log (Microsoft-Windows-Sysmon/Operational)
└── Wazuh Agent ships events → Manager → rule group: sysmon
Wireshark (Windows)
├── hooks into physical Wi-Fi NIC at Layer 2
├── captures all LAN traffic (frames → decoded up to Layer 7)
└── exports .pcap → copied to WSL2 for Zeek analysis
Zeek (WSL2 — pcap replay mode)
├── reads .pcap files from ~/zeek-analysis/
├── produces JSON structured logs: conn, dns, http, ssl, weird, x509, files
└── logs shipped via Logstash → OpenSearch zeek-* index
Logstash
├── input 1: /var/ossec/logs/alerts/alerts.json → wazuh-alerts-4.x-* index
├── input 2: ~/zeek-analysis/.log (JSON) → zeek- index
└── uses logstash-output-opensearch plugin (Filebeat incompatible with WSL2)
Wazuh Dashboard
├── queries OpenSearch for wazuh-alerts-* and zeek-* data
├── renders MITRE ATT&CK, FIM, Sysmon, Compliance, Vulnerability panels
└── accessible at https://172.23.201.127
TheHive (on-demand Docker)
├── start: cd ~/thehive && docker-compose up -d
├── tracks cases, observables (IPs, hashes, domains), analyst tasks
└── stop: docker-compose down (RAM conservation)
```
---

## OSI Layer Coverage

| OSI Layer | Tool Monitoring It | Status |
|---|---|---|
| Layer 7 — Application | Wazuh (Windows Event Logs), Zeek (HTTP/DNS), Sysmon (process behavior) | ✅ Active |
| Layer 6 — Presentation | Zeek (ssl.log — TLS certs, cipher suites), Wazuh (SSL rules) | ✅ Active |
| Layer 5 — Session | Zeek (conn.log — session tracking) | ✅ Active |
| Layer 4 — Transport | Zeek (TCP/UDP — ports, duration, bytes) | ✅ Active |
| Layer 3 — Network | Zeek (IP traffic), Wazuh (firewall rules) | ✅ Active |
| Layer 2 — Data Link | Wireshark (raw frame capture on physical NIC) | ✅ Active |
| Endpoint (all layers) | Wazuh Agent + Sysmon (Windows 11) | ✅ Active |

---

## Why WSL2 Instead of a Dedicated VM?

| Factor | VM-based Wazuh | WSL2 Wazuh (current) |
|---|---|---|
| RAM cost | Fixed 4–6GB | Dynamic — shares with Windows |
| Boot time | ~2 min | ~10 seconds |
| Dashboard access | VM IP only | localhost or WSL2 IP |
| File access | Separate filesystem | `/mnt/c/` bridges Windows |
| Attack lab coexistence | Hard on 12GB RAM | Comfortable — leaves 4GB for Kali |

---

## Why Zeek Runs in pcap Mode (Not Live)

WSL2 uses a virtual network adapter — it cannot access the physical NIC directly.
Physical Wi-Fi NIC (Windows)
│
Windows Hyper-V
│
Virtual Switch (vEthernet WSL)
│
WSL2 eth0 ← Zeek can only see THIS (WSL2-only traffic)

**Solution:** Capture on Windows (Wireshark has real NIC access) → export .pcap → analyze in Zeek on WSL2. This is standard threat hunting workflow used by professional analysts.

---

## Known Limitations

| Limitation | Impact | Workaround |
|---|---|---|
| WSL2 IP changes on reboot | FIXED | Static IP via `/etc/wsl.conf` boot command |
| Filebeat incompatible with WSL2 | Cannot use official pipeline | Replaced with Logstash + OpenSearch plugin |
| Zeek cannot do live capture on WSL2 | No real-time network monitoring | Wireshark → pcap → Zeek workflow |
| TheHive RAM usage | Too heavy to run always-on | On-demand via docker-compose up/down |
| Single Windows agent | No Linux endpoint | Kali VM agent planned for red team phase |

---

*Cerveaux Labs — Home SOC Operations*
*github.com/ellianima/home-soc-lab*
*Last updated: April 4, 2026*
