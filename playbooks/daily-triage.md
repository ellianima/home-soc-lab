# Daily Triage — Home SOC Operations Lab

**Analyst:** ellianima | **Lab:** Cerveaux Labs | **Updated:** 2026-04-03

---

## Morning Startup Checklist
```bash
# 1. Verify all services auto-started correctly
sudo systemctl status wazuh-manager wazuh-indexer wazuh-dashboard logstash | grep Active

# If Logstash is inactive (started before Indexer was ready):
sudo systemctl start logstash

# 2. Verify all services active
sudo systemctl status wazuh-manager wazuh-indexer wazuh-dashboard logstash | grep Active

# 3. Verify alerts are flowing to indexer
curl -k -u admin:admin "https://172.23.201.127:9200/wazuh-alerts-*/_count"

# 4. Open Wazuh dashboard
# Browser → https://172.23.201.127 or https://localhost
# Check: active agents, overnight alerts, severity levels

# 5. Push daily findings to GitHub (end of session)
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

# Restart all (correct order — always indexer first)
sudo systemctl restart wazuh-indexer
sudo systemctl restart wazuh-manager
sudo systemctl restart wazuh-dashboard
sudo systemctl restart logstash

# Verify alert pipeline is working
curl -k -u admin:admin "https://172.23.201.127:9200/wazuh-alerts-*/_count"

# Live alert monitoring from terminal
sudo tail -f /var/ossec/logs/alerts/alerts.json | grep -i "syscheck\|mitre\|authentication"
```

**Dashboard:** `https://172.23.201.127` or `https://localhost`
**Login:** `admin / admin`

> ✅ WSL2 IP is statically assigned via `/etc/wsl.conf` boot command. Persistent across reboots.

---

## Known Pipeline Notes

| Component | Status | Notes |
|---|---|---|
| Wazuh Manager | ✅ Active | Runs in WSL2 Ubuntu 22.04 |
| Wazuh Indexer | ✅ Active | OpenSearch 2.19.4 |
| Wazuh Dashboard | ✅ Active | Access via browser |
| Logstash | ✅ Active | Replaces Filebeat (WSL2 incompatible) |
| Filebeat | ❌ Disabled | pthread crash on WSL2 — replaced by Logstash |
| Zeek | 🔜 Planned | Network traffic analysis |
| TheHive | 🔜 Planned | Incident case management |

---

## Zeek Network Capture

> 🔜 **NOT SETUP YET** — commands below are for when Zeek is deployed
```bash
# Start capturing
sudo /opt/zeek/bin/zeek -i eth0 -C

# DNS log — who queried what domain
sudo /opt/zeek/bin/zeek-cut ts id.orig_h query answers < /opt/zeek/logs/current/dns.log

# Connection log — all LAN connections
sudo /opt/zeek/bin/zeek-cut ts id.orig_h id.resp_h id.resp_p proto < /opt/zeek/logs/current/conn.log

# HTTP traffic
sudo cat /opt/zeek/logs/current/http.log

# Long duration connections (potential C2 beaconing)
sudo /opt/zeek/bin/zeek-cut id.orig_h id.resp_h duration < /opt/zeek/logs/current/conn.log | sort -k3 -rn | head -20

# Anomalies
sudo cat /opt/zeek/logs/current/weird.log
```

---

## TheHive Incident Management

> 🔜 **NOT SETUP YET** — commands below are for when TheHive is deployed
```bash
# Start TheHive
sudo docker start thehive-thehive-1

# Stop TheHive (save RAM when not in use)
sudo docker stop thehive-thehive-1

# Check running
sudo docker ps

# View logs
sudo docker logs -f thehive-thehive-1
```

**Dashboard:** `http://localhost:9000`
**Login:** `admin@thehive.local / secret`

---

## IOC Investigation Workflow

When a suspicious domain, IP, hash, or file is found:
```bash
# DNS lookup
nslookup suspicious-domain.com
dig suspicious-domain.com

# Quick WHOIS
whois suspicious-domain.com
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

**Verdict decision:**
- Known ad network / CDN / cloud service → False Positive
- Unknown domain, low reputation, recently registered → Investigate further
- Confirmed malicious on VirusTotal → True Positive, escalate

---

## Wazuh DQL Quick Reference
```
# High severity alerts
rule.level >= 7

# MITRE-tagged alerts only
rule.mitre.id: *

# Specific technique
rule.mitre.id: T1098

# Agent 001 (Windows)
agent.id: 001

# Authentication failures
rule.groups: authentication_failed

# FIM events
rule.groups: syscheck

# Vulnerability alerts
rule.groups: vulnerability-detector

# SCA alerts
rule.groups: sca
```

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

# Check RAM
free -h

# Check disk space
df -h
```

---

## Linux Log & File Commands
```bash
# Read a file
cat filename.log

# Read with scroll
less filename.log

# Search inside file
grep "keyword" filename.log

# Case-insensitive search
grep -i "keyword" filename.log

# Show last 20 lines
tail -20 filename.log

# Follow live updates
tail -f filename.log

# List files with permissions
ls -lh

# Find a file
find / -name "filename" 2>/dev/null
```

---

## Git — GitHub Updates
```bash
# Navigate to lab folder
cd ~/home-soc-lab

# Stage all changes
git add .

# Commit with message
git commit -m "Daily log: 2026-04-03 — 3 alerts triaged, 1 true positive"

# Push to GitHub
git push origin main

# Check status
git status

# View commit history
git log --oneline
```

---

## Windows PowerShell — Wazuh Agent
```powershell
# Check agent status
Get-Service WazuhSvc

# Start agent
NET START WazuhSvc

# Stop agent
NET STOP WazuhSvc

# Restart agent
Restart-Service WazuhSvc

# Create FIM test file
New-Item -Path "C:\Users\Public\fim_test.txt" -ItemType File

# Check active connections
netstat -ano

# Check running processes
Get-Process
```

---

## Lab Network Reference

| Device | IP | Role | Agent Name |
|---|---|---|---|
| Ubuntu 22.04 (WSL2) | 172.23.201.127 | Wazuh Manager + Logstash | — |
| Windows 11 | 172.23.192.1 | Wazuh Agent + Endpoint | Candy-Cat-Silly-Fun |

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
[What you found, what tools you used, what IPs/domains/hashes you checked]

## Action Taken
[What you did — escalate, close, document]

## Lessons Learned
[What this taught you]
```
---

## Troubleshooting Quick Reference
```bash
# Logstash not indexing alerts?
curl -k -u admin:admin "https://172.23.201.127:9200/wazuh-alerts-*/_count"
sudo journalctl -u logstash --no-pager | tail -20

# Wazuh manager not starting?
sudo /var/ossec/bin/wazuh-control info
sudo tail -20 /var/ossec/logs/ossec.log

# Check MITRE database health
sudo sqlite3 /var/ossec/var/db/mitre.db ".tables"

# Verify WSL2 static IP is assigned correctly
hostname -I
# Should always show 172.23.201.127
# If not — check /etc/wsl.conf boot command

# Indexer health
curl -k -u admin:admin https://172.23.201.127:9200/_cluster/health
```

---

*Cerveaux Labs — Home SOC Operations*
*Analyst: ellianima*
*github.com/ellianima/home-soc-lab*
