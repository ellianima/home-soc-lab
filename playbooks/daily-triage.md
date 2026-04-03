# Daily Operations — Home SOC Operations Lab

**Analyst:** ellianima | **Lab:** Cerveaux Labs | **Updated:** 2026-03-26

---

## Morning Startup Checklist

```bash
# 1. Check RAM — must have 2GB+ available
free -h

# 2. Verify all Wazuh services are active
sudo systemctl status wazuh-manager | grep Active
sudo systemctl status wazuh-indexer | grep Active
sudo systemctl status wazuh-dashboard | grep Active

# 3. Open Wazuh dashboard
# Firefox → https://localhost
# Check: active agents, overnight alerts, severity levels

# 4. Start Zeek network capture (run 10 min, then Ctrl+C) [ ZEEK NOT SETUP YET ]
sudo /opt/zeek/bin/zeek -i eth0 -C

# 5. Analyze DNS log for suspicious domains
sudo /opt/zeek/bin/zeek-cut ts id.orig_h query answers < /opt/zeek/bin/dns.log

# 6. Check for network anomalies
sudo cat /opt/zeek/bin/weird.log

# 7. Push daily findings to GitHub
cd ~/home-soc-lab && git add . && git commit -m "Daily: $(date +%Y-%m-%d)" && git push origin main
```

---

## Wazuh Service Management

```bash
# Check status
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-indexer
sudo systemctl status wazuh-dashboard

# Restart if broken
sudo systemctl restart wazuh-manager
sudo systemctl restart wazuh-indexer
sudo systemctl restart wazuh-dashboard
```

**Dashboard:** `https://localhost`  
**Login:** admin / (saved from installation)

---

## Zeek Network Capture

```bash [ ZEEK NOT SETUP YET ]
# Start capturing on eth0
sudo /opt/zeek/bin/zeek -i eth0 -C

# Read DNS log (who queried what domain)
sudo /opt/zeek/bin/zeek-cut ts id.orig_h query answers < /opt/zeek/bin/dns.log

# Read connection log (all connections on LAN)
sudo /opt/zeek/bin/zeek-cut ts id.orig_h id.resp_h < /opt/zeek/bin/conn.log

# Read HTTP log (browser traffic)
sudo cat /opt/zeek/bin/http.log

# Read SSL log (encrypted connections)
sudo cat /opt/zeek/bin/ssl.log

# Check anomalies
sudo cat /opt/zeek/bin/weird.log
```

**Log location:** `/opt/zeek/bin/` (when run from that directory)

---

## TheHive Incident Management

```bash [ THEHIVE NOT SETUP YET ]
# Start TheHive (only when documenting incidents)
sudo docker start thehive-thehive-1

# Stop TheHive (to save RAM)
sudo docker stop thehive-thehive-1

# Check if running
sudo docker ps

# View startup logs
sudo docker logs -f thehive-thehive-1
```

**Dashboard:** `http://localhost:9000`  
**Login:** `admin@thehive.local` / `secret`

---

## IOC Investigation Workflow

When a suspicious domain, IP, or hash is found:

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
| VirusTotal | virustotal.com | IP/domain/hash reputation |
| urlscan.io | urlscan.io | URL behavior analysis |
| crt.sh | crt.sh | Certificate history |
| Shodan | shodan.io | Infrastructure lookup |
| AbuseIPDB | abuseipdb.com | IP abuse reports |

**Verdict decision:**

- Known ad network → False Positive
- Known CDN/cloud → False Positive
- Unknown domain, low reputation, recent registration → Investigate further
- Confirmed malicious on VirusTotal → True Positive, escalate

---

## Network Commands

```bash
# Check your IP
ip a

# Scan LAN for live devices
nmap -sn 172.23.201.127/20

# Scan open ports on a device
nmap -sV (insert ip address)

# Test connectivity
ping 192.168.1.8

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

# Read with scroll (q to quit)
less filename.log

# Search inside a file
grep "keyword" filename.log

# Search case-insensitive
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
# Go to lab folder
cd ~/home-soc-lab

# Stage all changes
git add .

# Commit with message
git commit -m "Daily log: 2026-03-26 — 3 alerts triaged, 1 true positive"

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
```

---

## Lab Network Reference

| Device     | IP             | Role        | Agent Name          |
| ---------- | -------------- | ----------- | ------------------- |
| Ubuntu VM  | 172.23.201.127 | SOC Manager | —                   |
| Windows 11 | 172.23.192.1   | Wazuh Agent | Candy-Cat-Silly-Fun |

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

| Time  | Event           |
| ----- | --------------- |
| HH:MM | [What happened] |

## Investigation

[What you found, what tools you used, what IPs/domains you checked]

## Action Taken

[What you did — escalate, close, document]

## Lessons Learned

[What this taught you]
```

## insert here
