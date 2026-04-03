# Daily Triage — Home SOC Operations Lab

**Analyst:** ellianima | **Lab:** Cerveaux Labs | **Updated:** 2026-04-03

---

## Morning Startup Checklist
```bash
# 1. Start all WSL2 services
sudo systemctl start wazuh-manager
sudo systemctl start wazuh-indexer
sudo systemctl start wazuh-dashboard
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

> ⚠️ WSL2 IP may change on reboot. Verify with `hostname -I`

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
