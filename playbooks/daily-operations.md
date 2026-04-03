# 🛡️ SOC Analyst L1 — Daily Operations Playbook

**Analyst:** ellianima
**Lab:** Cerveaux Labs Home SOC
**Last updated:** 2026-03-31

---

## The Core Loop

Every shift, every day, without exception:

```
Monitor → Detect → Triage → Investigate → Escalate or Close → Document
```

That's the job. Everything else is detail around this loop.

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

If an agent went silent overnight — **that IS the first incident of the day.** A dead agent = a blind spot. Investigate why before anything else.

Your agents:
| Agent | Host | IP | Expected status |
|-------|------|----|-----------------|
| 000 | ubuntu (manager) | 127.0.0.1 | Active/Local |
| 001 | Candy-Cat-Silly-Fun | 172.23.201.127 | Active |

### Step 3 — Check threat intel feeds

Spend 5 minutes scanning for anything new that affects your environment.

| Source                                                                   | What to check                 |
| ------------------------------------------------------------------------ | ----------------------------- |
| [CISA KEV](https://www.cisa.gov/known-exploited-vulnerabilities-catalog) | New exploited vulnerabilities |
| [AbuseIPDB](https://www.abuseipdb.com)                                   | Trending malicious IPs        |
| [OTX AlienVault](https://otx.alienvault.com)                             | Active campaigns              |
| [MITRE ATT&CK](https://attack.mitre.org)                                 | New technique additions       |

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
5. **Zeek conn.log** — any unusual outbound connections overnight?
6. **Zeek dns.log** — any suspicious domain queries?

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
5. Is there corroborating evidence in Zeek or Sysmon?
```

Question 5 is the most important. One data point is a signal. Three correlated data points is a true positive.

### Correlation matrix

| Wazuh fires  | + Zeek shows        | + Sysmon shows  | = Verdict                    |
| ------------ | ------------------- | --------------- | ---------------------------- |
| Auth failure | Outbound connection | New process     | True Positive — escalate     |
| Auth failure | Nothing unusual     | Nothing         | Likely FP — investigate user |
| User created | Lateral connection  | cmd.exe spawned | True Positive — escalate     |
| File changed | Nothing             | Nothing         | Verify manually              |

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
    ├─► Cross-reference with Zeek + Sysmon
    │       │
    │       ├─► Corroborated → TRUE POSITIVE
    │       │         Open TheHive case
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

Use these when an alert looks suspicious and needs deeper analysis.

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

| Tool       | When to use                          | What it gives you            |
| ---------- | ------------------------------------ | ---------------------------- |
| VirusTotal | Any IP, hash, domain, URL            | Multi-engine reputation scan |
| AbuseIPDB  | Suspicious IP in Zeek/Wazuh          | Abuse history + confidence % |
| CyberChef  | Obfuscated strings, encoded payloads | Decode/deobfuscate anything  |
| Any.run    | Suspicious file or process           | Live sandbox execution       |
| Shodan     | External IP from Zeek logs           | What services it's running   |

### Zeek quick reference

```bash
# Check recent connections
cat /opt/zeek/logs/current/conn.log | zeek-cut id.orig_h id.resp_h id.resp_p proto duration | head -50

# Find DNS queries
cat /opt/zeek/logs/current/dns.log | zeek-cut query answers | head -50

# Check HTTP traffic
cat /opt/zeek/logs/current/http.log | zeek-cut host uri user_agent | head -50

# Find long duration connections (potential beaconing)
cat /opt/zeek/logs/current/conn.log | zeek-cut id.orig_h id.resp_h duration | sort -k3 -rn | head -20
```

### Wazuh DQL quick reference

```
# All high severity alerts today
rule.level >= 7

# MITRE-tagged alerts only
rule.mitre.id: *

# Specific technique
rule.mitre.id: T1098

# Specific agent
agent.id: 003

# Authentication failures
rule.groups: authentication_failed

# FIM events
rule.groups: syscheck
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
- First-pass investigation (Wazuh + Zeek + OSINT)
- IOC collection and documentation
- Case creation in TheHive
- Shift handover notes

### L2 handles (escalate these)

- Deep malware analysis and reverse engineering
- Threat hunting (proactive search for hidden threats)
- Incident response lead
- Custom detection rule creation
- Memory forensics

### Escalation trigger — escalate immediately if

```
Level 12+ alert fires
Lateral movement detected (connections between internal hosts)
Ransomware indicators (mass file changes in FIM)
C2 beacon pattern in Zeek (regular interval connections to external IP)
Privilege escalation to SYSTEM/root detected
New service or scheduled task created outside business hours
```

---

## Your Home Lab Daily Checklist

```
MORNING
[ ] Read yesterday's daily-writeup.md
[ ] Verify all agents active in Wazuh
[ ] Check Level 12+ alerts — any?
[ ] Scan threat intel (CISA KEV, OTX)
[ ] Check MITRE panel — new tactics?
[ ] Check FIM — unexpected file changes?
[ ] Check Zeek conn.log — unusual connections?

DURING SESSION
[ ] Work alerts Level 7+ in priority order
[ ] Cross-reference every suspicious alert with Zeek + Sysmon
[ ] Run IOCs through VirusTotal / AbuseIPDB
[ ] Document every alert touched (FP or TP)

END OF SESSION
[ ] Write daily-writeup.md entry
[ ] Update open TheHive cases
[ ] Write shift handover note
[ ] git add . && git commit -m "daily ops [date]" && git push
```

---

_Cerveaux Labs — Home SOC Operations_
_Analyst: ellianima_
_github.com/ellianima/home-soc-lab_
