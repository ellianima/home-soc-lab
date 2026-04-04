# Daily Triage — Home SOC Operations Lab

**Analyst:** ellianima | **Lab:** Cerveaux Labs | **Updated:** 2026-04-04

---

## Morning Startup Checklist
```bash
# 1. Verify all services auto-started correctly
sudo systemctl status wazuh-manager wazuh-indexer wazuh-dashboard logstash | grep Active

# If Logstash is inactive (started before Indexer was ready):
sudo systemctl start logstash

# 2. Verify alerts are flowing
curl -k -u admin:admin "https://172.23.201.127:9200/wazuh-alerts-*/_count"
curl -k -u admin:admin "https://172.23.201.127:9200/zeek-*/_count"

# 3. Run Wireshark capture on Windows (Wi-Fi interface)
# Save to: C:\Users\elli\Desktop\JOB\PHASE A\SOC ANALYST L1\pcap folder\

# 4. Analyze capture with Zeek
cd ~/zeek-analysis
cp "/mnt/c/Users/elli/Desktop/JOB/PHASE A/SOC ANALYST L1/pcap folder/[capture].pcap" .
zeek -C -r [capture].pcap /opt/zeek/share/zeek/site/local.zeek

# 5. Open Wazuh dashboard
# Browser → https://172.23.201.127
# Check: active agents, overnight alerts, severity levels

# 6. Push daily findings to GitHub (end of session)
cd ~/home-soc-lab && git add . && git commit -m "Daily: $(date +%Y-%m-%d)" && git push origin main
```

---

## Wazuh Service Management
```bash
# Check status
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-indexer
sudo systemctl status wazuh-dashboard
sudo systemctl status logstash

# Restart all (always indexer first)
sudo systemctl restart wazuh-indexer
sudo systemctl restart wazuh-manager
sudo systemctl restart wazuh-dashboard
sudo systemctl restart logstash

# Verify alert pipeline
curl -k -u admin:admin "https://172.23.201.127:9200/wazuh-alerts-*/_count"

# Live alert monitoring
sudo tail -f /var/ossec/logs/alerts/alerts.json | grep -i "syscheck\|mitre\|authentication\|sysmon"
```

**Dashboard:** `https://172.23.201.127`
**Login:** `admin / admin`

> ✅ WSL2 IP statically assigned via `/etc/wsl.conf` boot command. Persistent across reboots.

---

## Known Pipeline Notes

| Component | Status | Notes |
|---|---|---|
| Wazuh Manager | ✅ Active | Runs in WSL2 Ubuntu 22.04 |
| Wazuh Indexer | ✅ Active | OpenSearch 2.19.4 |
| Wazuh Dashboard | ✅ Active | Access via browser |
| Logstash | ✅ Active | Replaces Filebeat (WSL2 incompatible) |
| Filebeat | ❌ Disabled | pthread crash on WSL2 — replaced by Logstash |
| Zeek | ✅ Active | pcap analysis mode — feed via Wireshark captures |
| Sysmon | ✅ Active | SwiftOnSecurity config — deep Windows telemetry |
| TheHive | ✅ Operational | Docker — start on demand only |

---

## Zeek Network Analysis

> Zeek runs in pcap replay mode — capture traffic on Windows via Wireshark, analyze in WSL2.
> Live capture not possible on WSL2 (virtual NIC — no access to physical NIC).
```bash
# Analyze a capture
cd ~/zeek-analysis
cp "/mnt/c/Users/elli/Desktop/JOB/PHASE A/SOC ANALYST L1/pcap folder/[capture].pcap" .
zeek -C -r [capture].pcap /opt/zeek/share/zeek/site/local.zeek

# Logs produced:
# conn.log   — every TCP/UDP connection
# dns.log    — every DNS query
# http.log   — HTTP requests, URIs, user agents
# ssl.log    — SSL/TLS certificates and ciphers
# weird.log  — protocol violations and anomalies
# x509.log   — certificate details
# files.log  — files transferred over the network

# Quick triage — check anomalies first
cat weird.log | python3 -m json.tool

# Filter specific weird events
grep "active_connection_reuse\|TCP_seq\|ip_hdr_len_zero" weird.log

# Top destination IPs by connection count
cat conn.log | python3 -c "
import sys, json
from collections import Counter
c = Counter()
for line in sys.stdin:
    try:
        d = json.loads(line)
        c[d.get('id.resp_h','')] += 1
    except: pass
for ip, n in c.most_common(10): print(n, ip)
"

# Long duration connections (potential C2 beaconing)
cat conn.log | python3 -c "
import sys, json
for line in sys.stdin:
    try:
        d = json.loads(line)
        if float(d.get('duration', 0)) > 60:
            print(d.get('duration'), d.get('id.orig_h'), '→', d.get('id.resp_h'), d.get('id.resp_p'))
    except: pass
"

# DNS queries
cat dns.log | python3 -m json.tool | grep '"query"'

# Verify Zeek logs are in OpenSearch
curl -k -u admin:admin "https://172.23.201.127:9200/zeek-*/_count"
```

**Zeek index in Dashboard:** `zeek-*` → Discover → filter by `type: zeek`

---

## TheHive Incident Management

> TheHive runs on-demand only — start when opening a case, stop when done.
```bash
# Start TheHive
cd ~/thehive && docker-compose up -d
# Wait ~3 minutes for Cassandra to initialize

# Check status
docker-compose ps
# Both cassandra (healthy) and thehive (Up) should show

# Stop TheHive (save RAM when not investigating)
cd ~/thehive && docker-compose down

# View logs if TheHive fails to start
docker-compose logs thehive | tail -20
docker-compose logs cassandra | tail -20
```

**Dashboard:** `http://localhost:9000`
**Login:** `admin@thehive.local / secret`

### Opening a case in TheHive
New Case →
Title:       [short description of incident]
Severity:    Low / Medium / High / Critical
Tags:        [mitre technique, tool, protocol]
Description: [what happened, what you found]
Add Observables →
Type: IP    → suspicious IP address
Type: domain → queried domain
Type: hash  → file hash
Type: other → process name, file path
Add Tasks →
"Check VirusTotal for [IP]"
"Review Zeek conn.log for [timeframe]"
"Escalate to L2 with findings"

---

## IOC Investigation Workflow
```bash
# DNS lookup
nslookup suspicious-domain.com
dig suspicious-domain.com

# WHOIS
whois suspicious-ip-or-domain
```

**Browser tools:**

| Tool | URL | Use |
|---|---|---|
| VirusTotal | virustotal.com | IP/domain/hash/URL reputation |
| AbuseIPDB | abuseipdb.com | IP abuse reports + confidence % |
| urlscan.io | urlscan.io | URL behavior + screenshot |
| crt.sh | crt.sh | Certificate history |
| Shodan | shodan.io | Infrastructure exposure |
| OTX AlienVault | otx.alienvault.com | Threat intel context |
| Any.run | any.run | Live sandbox execution |
| CyberChef | gchq.github.io/CyberChef | Decode/deobfuscate strings |
| APNIC whois | search.apnic.net | IPv6 / ASN ownership |

**Verdict decision:**
- Known CDN / cloud / ISP infrastructure → False Positive
- Unknown domain, low reputation, recently registered → Investigate further
- Confirmed malicious on VirusTotal → True Positive → open TheHive case

---

## Wazuh DQL Quick Reference
rule.level >= 7
rule.mitre.id: *
rule.mitre.id: T1098
agent.id: 001
rule.groups: authentication_failed
rule.groups: syscheck
rule.groups: sysmon
rule.groups: vulnerability-detector
rule.groups: sca

---

## Network Commands
```bash
# Check WSL2 IP
hostname -I

# Scan LAN for live devices
nmap -sn 172.23.201.0/20

# Scan open ports on a device
nmap -sV <target-ip>

# Check connectivity
ping 172.23.192.1

# Check RAM (important — monitor Wazuh + Docker usage)
free -h

# Check disk space
df -h
```

---

## Linux Log & File Commands
```bash
cat filename.log
less filename.log
grep "keyword" filename.log
grep -i "keyword" filename.log
tail -20 filename.log
tail -f filename.log
ls -lh
find / -name "filename" 2>/dev/null
```

---

## Git — GitHub Updates
```bash
cd ~/home-soc-lab
git add .
git commit -m "Daily log: 2026-04-04 — Zeek deployed, TheHive operational"
git push origin main
git status
git log --oneline
```

---

## Windows PowerShell — Wazuh Agent + Sysmon
```powershell
# Wazuh agent
Get-Service WazuhSvc
NET START WazuhSvc
NET STOP WazuhSvc
Restart-Service WazuhSvc

# Sysmon status
Get-Service Sysmon64

# Update Sysmon config
cd "$env:TEMP\Sysmon"
.\Sysmon64.exe -c sysmonconfig.xml

# Wireshark capture
# Open Wireshark → select Wi-Fi interface → capture → File → Save As → .pcap

# Create FIM test file
New-Item -Path "C:\Users\Public\fim_test.txt" -ItemType File

# Check active connections
netstat -ano

# Check running processes
Get-Process
```

---

## Lab Network Reference

| Device | IP | Role | Agent |
|---|---|---|---|
| Ubuntu 22.04 (WSL2) | 172.23.201.127 | Wazuh Manager + Logstash + Zeek | — |
| Windows 11 | 172.23.192.1 | Wazuh Agent + Sysmon + Wireshark | Candy-Cat-Silly-Fun |

---

## Incident Report Template
```markdown
# Case #XXX — [Title]

**Date:** YYYY-MM-DD
**Analyst:** ellianima
**Agent:** [Agent name] ([IP])
**Severity:** Low / Medium / High / Critical
**Verdict:** True Positive / False Positive

## Summary
[1-2 sentence description of what triggered the alert]

## MITRE ATT&CK
- Tactic: [Tactic name]
- Technique: [TXXXX — Technique name]

## Timeline
| Time | Event |
|---|---|
| HH:MM | [What happened] |

## Investigation
[What you found, tools used, IPs/domains/hashes checked]
[Zeek findings — conn.log, dns.log, weird.log]
[Sysmon findings — process spawns, network connections]

## IOCs
- IP: x.x.x.x
- Domain: suspicious.com
- Hash: [md5/sha256]
- Process: [name + path]

## Action Taken
[Escalate / Close / Document / TheHive case opened]

## Lessons Learned
[What this taught you]
```

---

## Zeek Incident Note Template
```markdown
**ALERT:**   [log type + event name + count]
**FINDING:** [OSINT tool used + result]
**VERDICT:** Benign / Suspicious / Malicious
**REASON:**  [specific explanation]
```

---

## Troubleshooting Quick Reference
```bash
# Logstash not indexing alerts?
curl -k -u admin:admin "https://172.23.201.127:9200/wazuh-alerts-*/_count"
sudo journalctl -u logstash --no-pager | tail -20

# Zeek logs not in OpenSearch?
curl -k -u admin:admin "https://172.23.201.127:9200/zeek-*/_count"
sudo journalctl -u logstash --no-pager | grep zeek
# Fix: sudo usermod -aG ellianima logstash

# Zeek sincedb issue (logs not re-read after restart)
sudo rm /var/lib/logstash/plugins/inputs/file/.sincedb*
sudo systemctl restart logstash

# Wazuh manager not starting?
sudo /var/ossec/bin/wazuh-control info
sudo tail -20 /var/ossec/logs/ossec.log

# MITRE database health
sudo sqlite3 /var/ossec/var/db/mitre.db ".tables"

# WSL2 IP check
hostname -I
# Should always show 172.23.201.127

# Indexer cluster health
curl -k -u admin:admin https://172.23.201.127:9200/_cluster/health

# TheHive not starting?
cd ~/thehive && docker-compose logs thehive | tail -20
# Fix: wait for cassandra healthy, then docker-compose restart thehive

# Sysmon not showing in Wazuh?
# Verify ossec.conf has Microsoft-Windows-Sysmon/Operational localfile entry
# Restart-Service WazuhSvc in PowerShell as Admin
```

---

## Incidents Log

| Case | Date | Title | Severity | Verdict |
|---|---|---|---|---|
| IR-2026-001 | 2026-04-04 | active_connection_reuse — PLDT DNS Server (IPv6) | Low | Benign — ISP DNS resolver |

---

*Cerveaux Labs — Home SOC Operations*
*Analyst: ellianima*
*github.com/ellianima/home-soc-lab*
*Last updated: April 4, 2026*
