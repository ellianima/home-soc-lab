# 🛡️ SOC Analyst L1 — Daily Operations Playbook

**Analyst:** ellianima
**Lab:** Cerveaux Labs Home SOC
**Last updated:** 2026-04-03

---

## The Core Loop

Every shift, every day, without exception:

```
Monitor → Detect → Triage → Investigate → Escalate or Close → Document
```

That's the job. Everything else is detail around this loop.

---

## Lab Architecture

| Component | Role | IP |
|---|---|---|
| Ubuntu 22.04 (WSL2) | Wazuh Manager + Logstash | 172.23.201.127 (static — persistent across reboots) |
| Windows 11 | Wazuh Agent + Monitored Endpoint | 172.23.192.1 |

> ✅ WSL2 IP is statically assigned via `/etc/wsl.conf` boot command. Persistent across reboots.

### Starting the lab (after Windows reboot)

```bash
# All services auto-start on WSL2 boot
# Verify they're running:
sudo systemctl status wazuh-manager wazuh-indexer wazuh-dashboard logstash | grep Active

# If any service failed to start (e.g. Logstash started before Indexer was ready):
sudo systemctl start logstash

# Verify all services
sudo systemctl status wazuh-manager wazuh-indexer wazuh-dashboard logstash | grep Active
```

Dashboard: `https://172.23.201.127` or `https://localhost`
Credentials: `admin / admin`

---

## Phase 1 — Morning Brief (15–30 min)

### Step 1 — Read shift handover notes

Before opening any tool, read what the previous session left behind.

In a corporate SOC this is a ticketing system (ServiceNow, Jira).
In your home lab this is your `daily-writeup.md` from yesterday.

Questions to answer:

- What cases are still open?
- Was anything escalated that needs follow-up?
- What context do I need before touching today's alerts?

### Step 2 — Check agent health

Open Wazuh dashboard. Before looking at a single alert, verify every agent is alive.

```
Agents Summary → All agents showing Active?
Last keep alive → Is the timestamp recent (within last 30 seconds)?
```

If an agent went silent — **that IS the first incident of the day.** A dead agent = a blind spot. Investigate why before anything else.

Your agents:
| Agent | Host | IP | Expected status |
|---|---|---|---|
| 001 | Candy-Cat-Silly-Fun (Windows 11) | 172.23.192.1 | Active |

### Step 3 — Check threat intel feeds

Spend 5 minutes scanning for anything new that affects your environment.

| Source | What to check |
|---|---|
| [CISA KEV](https://www.cisa.gov/known-exploited-vulnerabilities-catalog) | New exploited vulnerabilities |
| [AbuseIPDB](https://www.abuseipdb.com) | Trending malicious IPs |
| [OTX AlienVault](https://otx.alienvault.com) | Active campaigns |
| [MITRE ATT&CK](https://attack.mitre.org) | New technique additions |

Ask: does any of this affect Windows 11, Wazuh 4.14.4, or anything on my network?

---

## Phase 2 — Alert Triage (Bulk of your shift)

### Wazuh priority order

Never start at Level 3 and work up. Always work top-down by severity.

```
Level 12–15  →  Drop everything. Investigate immediately.
Level 7–11   →  High priority. Work within the hour.
Level 4–6    →  Medium. Work through during shift.
Level 1–3    →  Low noise. Review in bulk at end of shift.
```

### Morning dashboard checks — in order

1. **Level 12+ alerts** — any? If yes, this becomes your entire focus
2. **Authentication failures** — brute force? Unusual account?
3. **MITRE ATT&CK panel** — any new tactics since yesterday?
4. **FIM recent events** — any unexpected file changes?
5. **Vulnerability Detection** — any new Critical/High findings?
6. **SCA score** — did the benchmark score change?

> 📝 Zeek network analysis coming soon — conn.log and dns.log checks will be added when Zeek is deployed.

---

## Phase 3 — Per-Alert Triage Loop

Repeat this for every alert you touch.

### The 5 questions

For every alert, answer these before doing anything else:

```
1. What rule triggered it, and what does that rule actually detect?
2. Which host/agent is the source?
3. What time did it fire — is that normal for this host?
4. Has this exact pattern fired before, or is this new behavior?
5. Is there corroborating evidence in FIM, Sysmon, or other rules?
```

Question 5 is the most important. One data point is a signal. Three correlated data points is a true positive.

### Correlation matrix

| Wazuh fires | + FIM shows | + Sysmon shows | = Verdict |
|---|---|---|---|
| Auth failure | Unexpected file change | New process | True Positive — escalate |
| Auth failure | Nothing unusual | Nothing | Likely FP — investigate user |
| User created | Startup folder modified | cmd.exe spawned | True Positive — escalate |
| File changed | Nothing | Nothing | Verify manually |

### Verdict decision tree

```
Alert fires
    │
    ├─► Is this a known pattern? (same rule, same source, happened before)
    │       │
    │       ├─► YES → Likely false positive
    │       │         Document reason → Close
    │       │         Consider rule tuning if recurring
    │       │
    │       └─► NO → Continue investigation
    │
    ├─► Cross-reference with FIM + Sysmon + Vulnerability data
    │       │
    │       ├─► Corroborated → TRUE POSITIVE
    │       │         Open TheHive case (when deployed)
    │       │         Document timeline + IOCs
    │       │         Escalate to L2 with findings
    │       │
    │       └─► Not corroborated → UNCLEAR
    │                 Document findings
    │                 Escalate to L2 with "needs further investigation"
    │
    └─► Document everything regardless of verdict
```

---

## Phase 4 — Investigation Tools

### IOC lookup flow

```
Suspicious IP/domain/hash found
        │
        ├─► VirusTotal → reputation + detections
        ├─► AbuseIPDB → abuse reports + confidence score
        ├─► Shodan → what's exposed on that IP?
        └─► OTX AlienVault → threat intel context
```

### Analysis tools

| Tool | When to use | What it gives you |
|---|---|---|
| VirusTotal | Any IP, hash, domain, URL | Multi-engine reputation scan |
| AbuseIPDB | Suspicious IP in alerts | Abuse history + confidence % |
| CyberChef | Obfuscated strings, encoded payloads | Decode/deobfuscate anything |
| Any.run | Suspicious file or process | Live sandbox execution |
| Shodan | External IP from logs | What services it's running |
| urlscan.io | Suspicious URL | Screenshot + behavior analysis |

### Wazuh DQL quick reference

```
# All high severity alerts today
rule.level >= 7

# MITRE-tagged alerts only
rule.mitre.id: *

# Specific technique
rule.mitre.id: T1098

# Specific agent
agent.id: 001

# Authentication failures
rule.groups: authentication_failed

# FIM events
rule.groups: syscheck

# Vulnerability alerts
rule.groups: vulnerability-detector
```

### Check alerts from WSL2 terminal

```bash
# Live alert monitoring
sudo tail -f /var/ossec/logs/alerts/alerts.json | python3 -m json.tool

# FIM events only
sudo tail -f /var/ossec/logs/alerts/alerts.json | grep -i "syscheck"

# High severity only (level 7+)
sudo tail -f /var/ossec/logs/alerts/alerts.json | grep -E '"level":[7-9]|"level":1[0-5]'

# Verify alert count in indexer
curl -k -u admin:admin "https://172.23.201.127:9200/wazuh-alerts-*/_count"
```

---

## Phase 5 — Documentation

This is where most junior analysts fail. Every alert you touch needs a record.

### False positive format

```markdown
**Alert:** [Rule ID] — [Description] — Level [N]
**Time:** [timestamp]
**Source:** [hostname / IP / process]
**Reason:** [Why this is benign — be specific]
**Verdict:** FALSE POSITIVE
**Action:** No escalation. [Tuning recommendation if recurring]
```

### True positive / incident report format

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

**Recommendation:**
- [what should happen next — L2 task]
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
- First-pass investigation (Wazuh + OSINT)
- IOC collection and documentation
- Case creation in TheHive (when deployed)
- Shift handover notes

### L2 handles (escalate these)
- Deep malware analysis and reverse engineering
- Threat hunting (proactive search for hidden threats)
- Incident response lead
- Custom detection rule creation
- Memory forensics

### Escalation triggers — escalate immediately if

```
Level 12+ alert fires
Lateral movement detected (connections between internal hosts)
Ransomware indicators (mass file changes in FIM)
Privilege escalation to SYSTEM/root detected
New service or scheduled task created outside business hours
Unknown process spawned by lsass.exe or svchost.exe
```

---

## Home Lab Daily Checklist

```
STARTUP
[ ] Verify WSL2 services (manager, indexer, dashboard, logstash)
[ ] Verify all services active
[ ] Confirm agent 001 (Candy-Cat-Silly-Fun) is online

MORNING
[ ] Read yesterday's daily-writeup.md
[ ] Verify all agents active in Wazuh
[ ] Check Level 12+ alerts — any?
[ ] Scan threat intel (CISA KEV, OTX)
[ ] Check MITRE panel — new tactics?
[ ] Check FIM — unexpected file changes?
[ ] Check Vulnerability Detection — new Critical/High?

DURING SESSION
[ ] Work alerts Level 7+ in priority order
[ ] Cross-reference every suspicious alert with FIM + Sysmon
[ ] Run IOCs through VirusTotal / AbuseIPDB
[ ] Document every alert touched (FP or TP)

END OF SESSION
[ ] Write daily-writeup.md entry
[ ] Update open cases
[ ] Write shift handover note
[ ] git add . && git commit -m "daily ops [date]" && git push
```

---

## Known Issues & Lab Notes

| Issue | Status | Workaround |
|---|---|---|
| WSL2 IP resets on reboot | FIXED  | Static IP via /etc/wsl.conf boot command |
| Filebeat crashes on WSL2 | Known — replaced | Using Logstash + OpenSearch plugin instead |
| Logstash needs Elasticsearch plugin installed | Known | Keep both `logstash-output-elasticsearch` and `logstash-output-opensearch` installed |
| `perf_event_paranoid` resets on WSL2 restart | Mitigated | Set in `/etc/sysctl.conf` and `/etc/wsl.conf` |
| Logstash inactive after reboot | FIXED | ExecStartPre=/bin/sleep 30 in logstash.service |

---

*Cerveaux Labs — Home SOC Operations*
*Analyst: ellianima*
*github.com/ellianima/home-soc-lab*
