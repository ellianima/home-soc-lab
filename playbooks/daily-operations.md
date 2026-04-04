# 🛡️ SOC Analyst L1 — Daily Operations Playbook

**Analyst:** ellianima
**Lab:** Cerveaux Labs Home SOC
**Last updated:** 2026-04-04

---

## The Core Loop

Every shift, every day, without exception:
Monitor → Detect → Triage → Investigate → Escalate or Close → Document

---

## Lab Architecture

| Component | Role | IP |
|---|---|---|
| Ubuntu 22.04 (WSL2) | Wazuh Manager + Logstash + Zeek | 172.23.201.127 |
| Windows 11 | Wazuh Agent + Sysmon + Wireshark | 172.23.192.1 |

> ✅ WSL2 IP statically assigned via `/etc/wsl.conf` boot command.

### Starting the lab
```bash
# All services auto-start on WSL2 boot
# Verify:
sudo systemctl status wazuh-manager wazuh-indexer wazuh-dashboard logstash | grep Active

# If Logstash failed (started before Indexer):
sudo systemctl start logstash

# TheHive — start on demand only
cd ~/thehive && docker-compose up -d

# TheHive — stop when done
cd ~/thehive && docker-compose down
```

Dashboard: `https://172.23.201.127`
TheHive: `http://localhost:9000`
Credentials Wazuh: `admin / admin`
Credentials TheHive: `admin@thehive.local / secret`

---

## Phase 1 — Morning Brief (15–30 min)

### Step 1 — Read shift handover notes

Before opening any tool, read yesterday's `daily-writeup.md`.

- What cases are still open?
- Was anything escalated that needs follow-up?
- What context do I need before touching today's alerts?

### Step 2 — Check agent health

Open Wazuh dashboard. Before looking at alerts, verify every agent is alive.
Agents Summary → All agents showing Active?
Last keep alive → Recent timestamp (within last 30 seconds)?

A dead agent = a blind spot. **That IS the first incident of the day.**

Your agents:

| Agent | Host | IP | Expected Status |
|---|---|---|---|
| 001 | Candy-Cat-Silly-Fun (Windows 11) | 172.23.192.1 | Active |

### Step 3 — Zeek Network Check
```bash
cd ~/zeek-analysis

# Copy latest capture from Windows
cp "/mnt/c/Users/elli/Desktop/JOB/PHASE A/SOC ANALYST L1/pcap folder/[capture].pcap" .

# Analyze
zeek -C -r [capture].pcap /opt/zeek/share/zeek/site/local.zeek

# Quick triage
grep "active_connection_reuse\|TCP_seq\|ip_hdr_len_zero" weird.log
cat dns.log | head -20
cat conn.log | sort -k9 -rn | head -10   # top connections by bytes
```

Key questions:
- Any IPs in `weird.log` that aren't ISP/CDN infrastructure?
- Any DNS queries to random-looking domains?
- Any long-duration connections to unknown external IPs?

### Step 4 — Check threat intel feeds

| Source | What to check |
|---|---|
| [CISA KEV](https://www.cisa.gov/known-exploited-vulnerabilities-catalog) | New exploited vulnerabilities |
| [AbuseIPDB](https://www.abuseipdb.com) | Trending malicious IPs |
| [OTX AlienVault](https://otx.alienvault.com) | Active campaigns |
| [MITRE ATT&CK](https://attack.mitre.org) | New technique additions |

---

## Phase 2 — Alert Triage

### Wazuh priority order
Level 12–15  →  Drop everything. Investigate immediately.
Level 7–11   →  High priority. Work within the hour.
Level 4–6    →  Medium. Work through during shift.
Level 1–3    →  Low noise. Review in bulk at end of shift.

### Morning dashboard checks — in order

1. **Level 12+ alerts** — any? If yes, full focus here
2. **Sysmon alerts** — any suspicious process spawns?
3. **Authentication failures** — brute force? Unusual account?
4. **MITRE ATT&CK panel** — new tactics since yesterday?
5. **FIM recent events** — unexpected file changes?
6. **Vulnerability Detection** — new Critical/High findings?
7. **Zeek index** (`zeek-*`) — anomalies in network layer?

---

## Phase 3 — Per-Alert Triage Loop

### The 5 questions

What rule triggered it, and what does it actually detect?
Which host/agent is the source?
What time did it fire — is that normal for this host?
Has this exact pattern fired before, or is this new behavior?
Is there corroborating evidence in FIM, Sysmon, Zeek, or other rules?


Question 5 is the most important. One data point is a signal.
Three correlated data points is a true positive.

### Correlation matrix

| Wazuh fires | + Sysmon shows | + Zeek shows | = Verdict |
|---|---|---|---|
| Auth failure | New process spawned | Outbound connection to unknown IP | True Positive — escalate |
| Auth failure | Nothing unusual | Nothing | Likely FP — investigate user |
| User created | Startup folder modified | DNS query to random domain | True Positive — escalate |
| File changed | Nothing | Nothing | Verify manually |
| PowerShell alert | cmd.exe spawned | Long duration external connection | True Positive — escalate |

### Verdict decision tree
```
Alert fires
│
├─► Known pattern? (same rule, same source, happened before)
│       ├─► YES → Likely FP → Document → Close
│       └─► NO → Continue investigation
│
├─► Cross-reference FIM + Sysmon + Zeek
│       ├─► Corroborated → TRUE POSITIVE
│       │         Open TheHive case
│       │         Document timeline + IOCs
│       │         Escalate to L2
│       │
│       └─► Not corroborated → UNCLEAR
│                 Document findings
│                 Escalate with "needs further investigation"
│
└─► Document everything regardless of verdict
```
---

## Phase 4 — Investigation Tools

### IOC lookup flow
Suspicious IP/domain/hash found
│
├─► VirusTotal    → reputation + detections
├─► AbuseIPDB     → abuse reports + confidence score
├─► whois/APNIC   → who owns this IP/ASN?
├─► Shodan        → what's exposed on that IP?
└─► OTX AlienVault → threat intel context

### Zeek quick reference
```bash
# All weird events
cat weird.log | python3 -m json.tool

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

# DNS queries only
cat dns.log | python3 -m json.tool | grep '"query"'

# Long duration connections (potential C2)
cat conn.log | python3 -c "
import sys, json
for line in sys.stdin:
    try:
        d = json.loads(line)
        if float(d.get('duration', 0)) > 60:
            print(d.get('duration'), d.get('id.orig_h'), '→', d.get('id.resp_h'), d.get('id.resp_p'))
    except: pass
"
```

### Wazuh DQL quick reference
rule.level >= 7
rule.mitre.id: *
rule.mitre.id: T1098
agent.id: 001
rule.groups: authentication_failed
rule.groups: syscheck
rule.groups: sysmon
rule.groups: vulnerability-detector

### Analysis tools

| Tool | When to use | What it gives you |
|---|---|---|
| VirusTotal | Any IP, hash, domain, URL | Multi-engine reputation scan |
| AbuseIPDB | Suspicious IP in alerts | Abuse history + confidence % |
| whois / APNIC | Unknown IP | ASN, ISP, country |
| CyberChef | Obfuscated strings | Decode/deobfuscate |
| Any.run | Suspicious file or process | Live sandbox execution |
| Shodan | External IP from logs | Exposed services |
| urlscan.io | Suspicious URL | Screenshot + behavior |

---

## Phase 5 — TheHive Case Management

Start TheHive only when opening a case:
```bash
cd ~/thehive && docker-compose up -d
# Wait ~3 minutes for full initialization
# Access: http://localhost:9000
```

### Opening a case
New Case →
Title:      [short description]
Severity:   Low / Medium / High / Critical
Tags:       [mitre technique, tool, protocol]
Description: [what happened, what you found]
Add Observables →
IP addresses involved
Domain names queried
File hashes
Process names
Add Tasks →
"Check VirusTotal for [IP]"
"Review Zeek conn.log for [timeframe]"
"Escalate to L2 with findings"

### Incident note format (inside TheHive case)
```markdown
**Incident ID:** IR-[YYYY]-[NNN]
**Date/Time:** [timestamp]
**Severity:** Critical / High / Medium / Low
**Agent:** [hostname]

**Summary:** [One sentence — what happened]

**Timeline:**
- [time] — [what happened]
- [time] — [what happened]

**IOCs:**
- IP: x.x.x.x
- Hash: [md5/sha256]
- Process: [name + path]
- File: [path]

**MITRE Techniques:**
- [T-number] — [name] — [tactic]

**Actions taken:**
- [what you did]

**Verdict:** [Benign / True Positive / Escalated]

**Recommendation:**
- [what should happen next]
```

Stop TheHive when done:
```bash
cd ~/thehive && docker-compose down
```

---

## Phase 6 — Documentation

### Zeek incident note format
```markdown
**ALERT:**   [what Zeek flagged — log type + event name + count]
**FINDING:** [what OSINT revealed — tool used + result]
**VERDICT:** [Benign / Suspicious / Malicious]
**REASON:**  [specific explanation]
```

### False positive format
```markdown
**Alert:** [Rule ID] — [Description] — Level [N]
**Time:** [timestamp]
**Source:** [hostname / IP / process]
**Reason:** [Why this is benign — be specific]
**Verdict:** FALSE POSITIVE
**Action:** No escalation. [Tuning recommendation if recurring]
```

### Shift handover note format
```markdown
## Shift Handover — [date] @ [time]

**Open cases:** [TheHive case IDs or "none"]
**Escalated:** [anything sent to L2?]
**Notable findings:** [anything next analyst should know]
**Rule tuning needed:** [any recurring FPs to suppress?]
**Next steps:** [anything outstanding]
```

---

## L1 Boundary — Where Your Job Ends

### L1 handles
- Alert triage and FP/TP classification
- First-pass investigation (Wazuh + Zeek + OSINT)
- IOC collection and documentation
- Case creation and management in TheHive
- Shift handover notes

### L2 handles (escalate these)
- Deep malware analysis and reverse engineering
- Threat hunting (proactive)
- Incident response lead
- Custom detection rule creation
- Memory forensics

### Escalation triggers
Level 12+ alert fires
Lateral movement detected (internal → internal connections)
Ransomware indicators (mass FIM changes)
Privilege escalation to SYSTEM/root
New service or scheduled task outside business hours
Unknown process spawned by lsass.exe or svchost.exe
Zeek: long duration outbound connection to unknown IP
Zeek: DNS queries to random-looking subdomains (possible tunneling)
Zeek: port 443 traffic with no ssl.log entry

---

## Daily Checklist
STARTUP
[ ] WSL2 services active (manager, indexer, dashboard, logstash)
[ ] Agent 001 (Candy-Cat-Silly-Fun) showing Active
[ ] Alert count > 0 in indexer
MORNING
[ ] Read yesterday's daily-writeup.md
[ ] Run Wireshark capture → analyze with Zeek
[ ] Check weird.log — any anomalies?
[ ] Check Wazuh Level 12+ alerts — any?
[ ] Check Sysmon alerts — suspicious process spawns?
[ ] Scan threat intel (CISA KEV, OTX)
[ ] Check MITRE panel — new tactics?
[ ] Check FIM — unexpected file changes?
DURING SESSION
[ ] Work alerts Level 7+ in priority order
[ ] Cross-reference suspicious alerts: Wazuh + Sysmon + Zeek
[ ] Run IOCs through VirusTotal / AbuseIPDB / whois
[ ] Document every alert touched (FP or TP)
[ ] Open TheHive case for True Positives
END OF SESSION
[ ] Write daily-writeup.md entry
[ ] Update open TheHive cases
[ ] Write shift handover note
[ ] Stop TheHive: docker-compose down
[ ] git add . && git commit -m "daily ops [date]" && git push

---

## Known Issues & Lab Notes

| Issue | Status | Workaround |
|---|---|---|
| WSL2 IP resets on reboot | ✅ Fixed | Static IP via `/etc/wsl.conf` boot command |
| Filebeat crashes on WSL2 | ✅ Replaced | Using Logstash + OpenSearch plugin |
| Logstash needs ES plugin installed | Known | Keep both output plugins installed |
| `perf_event_paranoid` resets | ✅ Mitigated | Set in `/etc/wsl.conf` boot command |
| Logstash inactive after reboot | ✅ Fixed | `ExecStartPre=/bin/sleep 30` in logstash.service |
| Zeek bad checksum warnings | Known — harmless | Use `zeek -C` flag for pcap analysis |
| Zeek logs not shipping to Logstash | ✅ Fixed | `sudo usermod -aG ellianima logstash` |
| TheHive exits on start | ✅ Fixed | Cassandra healthcheck in docker-compose |
| TheHive RAM usage | ✅ Managed | On-demand only — docker-compose up/down |

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
