# SOC Lab Setup Guide

Full reproducible setup documentation for the Home SOC Operations Lab.
Environment: WSL2 Ubuntu 22.04 on Windows 11 host, Kali Linux VM (attacker — planned).

---

## Prerequisites

- Windows 11 (host machine)
- WSL2 enabled with Ubuntu 22.04 LTS
- Wireshark 4.6.4 (Windows — packet capture)
- Oracle VirtualBox 7.x (for Kali attacker VM — planned)
- Minimum host RAM: 12GB (WSL2: 8GB allocated, Kali VM: 4GB planned)

---

## Architecture Overview
```
Windows 11 (Host — 12GB RAM)
├── WSL2 Ubuntu 22.04 (8GB allocated via .wslconfig)
│   ├── Wazuh Manager 4.14.4
│   ├── Wazuh Indexer 4.14.4 (OpenSearch)
│   ├── Wazuh Dashboard 4.14.4
│   ├── Logstash 7.17.29 (Wazuh + Zeek pipeline)
│   ├── Zeek 8.0.5 (pcap analysis)
│   └── TheHive 5.2.16 (Docker — on-demand)
├── Wazuh Agent 4.14.4 (Windows self-monitoring)
├── Sysmon 15.20 (SwiftOnSecurity config)
├── Wireshark 4.6.4 (live capture → .pcap export)
└── VirtualBox
└── Kali Linux VM — 4GB RAM [🔜 Planned — attacker only]
```
---


---

## Phase 1 — WSL2 Setup

### 1.1 Install WSL2
```powershell
wsl --install --no-distribution
# Restart Windows after install
```

### 1.2 Enable Virtual Machine Platform
```powershell
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
# Restart Windows
```

### 1.3 Enable BIOS Virtualization

Lenovo LOQ 15: F2 on boot → Configuration → Intel Virtualization Technology → Enabled
Also enable: Intel VT-d
Save with F10

### 1.4 Install Ubuntu 22.04
```powershell
wsl --install -d Ubuntu-22.04
# Set username: ellianima
```

### 1.5 Configure WSL2 Memory + Static IP

Create `C:\Users\elli\.wslconfig`:
```ini
[wsl2]
memory=8GB
swap=2GB
```

Edit `/etc/wsl.conf` inside WSL2:
```ini
[boot]
systemd=true
command = sysctl -w kernel.perf_event_paranoid=1 && ip addr add 172.23.201.127/20 dev eth0 2>/dev/null || true

[user]
default=ellianima
```

> ⚠️ Static IP is assigned on every boot via the boot command.
> WSL2's virtual NIC resets on restart — this is the fix.

---

## Phase 2 — Wazuh Installation (WSL2)

### 2.1 Update Ubuntu
```bash
sudo apt-get update && sudo apt-get upgrade -y
sudo apt-get install curl -y
```

### 2.2 Add Wazuh Repository
```bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg --no-default-keyring \
  --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import && \
  sudo chmod 644 /usr/share/keyrings/wazuh.gpg

echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] \
  https://packages.wazuh.com/4.x/apt/ stable main" | \
  sudo tee -a /etc/apt/sources.list.d/wazuh.list

sudo apt-get update
```

### 2.3 Install Wazuh Manager
```bash
sudo apt-get install wazuh-manager -y
sudo systemctl daemon-reload
sudo systemctl enable wazuh-manager
sudo systemctl start wazuh-manager
```

### 2.4 Generate SSL Certificates
```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-certs-tool.sh
curl -sO https://packages.wazuh.com/4.14/config.yml
nano config.yml
# Set all IPs to 172.23.201.127
sudo bash wazuh-certs-tool.sh -A
```

### 2.5 Install Wazuh Indexer
```bash
sudo apt-get install wazuh-indexer -y

sudo mkdir /etc/wazuh-indexer/certs
sudo cp ~/wazuh-certificates/node-1.pem /etc/wazuh-indexer/certs/indexer.pem
sudo cp ~/wazuh-certificates/node-1-key.pem /etc/wazuh-indexer/certs/indexer-key.pem
sudo cp ~/wazuh-certificates/root-ca.pem /etc/wazuh-indexer/certs/root-ca.pem
sudo cp ~/wazuh-certificates/admin.pem /etc/wazuh-indexer/certs/admin.pem
sudo cp ~/wazuh-certificates/admin-key.pem /etc/wazuh-indexer/certs/admin-key.pem
sudo bash -c 'chmod 400 /etc/wazuh-indexer/certs/*'
sudo chown -R wazuh-indexer:wazuh-indexer /etc/wazuh-indexer/certs

sudo systemctl daemon-reload
sudo systemctl enable wazuh-indexer
sudo systemctl start wazuh-indexer
sudo /usr/share/wazuh-indexer/bin/indexer-security-init.sh
```

### 2.6 Install Wazuh Dashboard
```bash
sudo apt-get install wazuh-dashboard -y

sudo mkdir /etc/wazuh-dashboard/certs
sudo cp ~/wazuh-certificates/dashboard.pem /etc/wazuh-dashboard/certs/dashboard.pem
sudo cp ~/wazuh-certificates/dashboard-key.pem /etc/wazuh-dashboard/certs/dashboard-key.pem
sudo cp ~/wazuh-certificates/root-ca.pem /etc/wazuh-dashboard/certs/root-ca.pem
sudo bash -c 'chmod 400 /etc/wazuh-dashboard/certs/*'
sudo chown -R wazuh-dashboard:wazuh-dashboard /etc/wazuh-dashboard/certs

# Fix dashboard config
sudo nano /etc/wazuh-dashboard/opensearch_dashboards.yml
# Set: opensearch.hosts: https://172.23.201.127:9200

sudo systemctl daemon-reload
sudo systemctl enable wazuh-dashboard
sudo systemctl start wazuh-dashboard
```

**Dashboard:** `https://172.23.201.127`
**Login:** `admin / admin`

---

## Phase 3 — Logstash Alert Pipeline

> ⚠️ Filebeat crashes on WSL2 due to pthread threading restrictions.
> Logstash with OpenSearch plugin is the replacement.

### 3.1 Install Logstash
```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | \
  sudo gpg --dearmor -o /usr/share/keyrings/elastic-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/elastic-keyring.gpg] \
  https://artifacts.elastic.co/packages/7.x/apt stable main" | \
  sudo tee /etc/apt/sources.list.d/elastic-7.x.list

sudo apt-get update
sudo apt-get install logstash -y
```

### 3.2 Install OpenSearch Plugin
```bash
sudo /usr/share/logstash/bin/logstash-plugin install logstash-output-opensearch
# Do NOT remove logstash-output-elasticsearch — internal dependency
```

### 3.3 Configure Wazuh Pipeline
```bash
sudo curl -so /etc/filebeat/wazuh-template.json \
  https://raw.githubusercontent.com/wazuh/wazuh/v4.14.4/extensions/elasticsearch/7.x/wazuh-template.json

curl -k -u admin:admin -X PUT "https://172.23.201.127:9200/_template/wazuh" \
  -H 'Content-Type: application/json' \
  -d @/etc/filebeat/wazuh-template.json

sudo tee /etc/logstash/conf.d/wazuh.conf << 'EOF'
input {
  file {
    path => "/var/ossec/logs/alerts/alerts.json"
    codec => "json"
    type => "wazuh-alerts"
    start_position => "beginning"
  }
}
filter {
  if [type] == "wazuh-alerts" {
    date {
      match => ["timestamp", "ISO8601"]
      target => "@timestamp"
    }
  }
}
output {
  opensearch {
    hosts => ["https://172.23.201.127:9200"]
    index => "wazuh-alerts-4.x-%{+YYYY.MM.dd}"
    user => "admin"
    password => "admin"
    ssl => true
    ssl_certificate_verification => false
  }
}
EOF

sudo usermod -aG wazuh logstash
sudo /var/ossec/bin/wazuh-keystore -f indexer -k username -v admin
sudo /var/ossec/bin/wazuh-keystore -f indexer -k password -v admin
sudo systemctl enable logstash
sudo systemctl start logstash
```

### 3.4 Fix Logstash Startup Timing
```bash
sudo nano /etc/systemd/system/logstash.service
# Add under [Service]:
# ExecStartPre=/bin/sleep 30
sudo systemctl daemon-reload
```

### 3.5 Verify
```bash
curl -k -u admin:admin "https://172.23.201.127:9200/wazuh-alerts-*/_count"
# Should return count > 0
```

---

## Phase 4 — Windows Agent + Sysmon

### 4.1 Deploy Wazuh Agent

Dashboard → Server Management → Endpoints Summary → Deploy new agent
OS: Windows | Architecture: x86_64 | Server: 172.23.201.127
Run generated PowerShell command as Administrator.

### 4.2 Configure FIM

Edit `C:\Program Files (x86)\ossec-agent\ossec.conf` as Administrator.
Add inside `<syscheck>`:
```xml
<directories realtime="yes" report_changes="yes">C:\Users\Public</directories>
<directories realtime="yes">C:\Windows\System32</directories>
<directories realtime="yes">C:\ProgramData</directories>
```

### 4.3 Install Sysmon
```powershell
# Run as Administrator
Invoke-WebRequest -Uri "https://download.sysinternals.com/files/Sysmon.zip" -OutFile "$env:TEMP\Sysmon.zip"
Expand-Archive "$env:TEMP\Sysmon.zip" -DestinationPath "$env:TEMP\Sysmon"
cd "$env:TEMP\Sysmon"

# Download SwiftOnSecurity config
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml" -OutFile "$env:TEMP\Sysmon\sysmonconfig.xml"

# Install
.\Sysmon64.exe -accepteula -i sysmonconfig.xml

# If already installed, update config:
.\Sysmon64.exe -c sysmonconfig.xml
```

### 4.4 Configure Wazuh to Read Sysmon

Add to `C:\Program Files (x86)\ossec-agent\ossec.conf`:
```xml
<localfile>
    <location>Microsoft-Windows-Sysmon/Operational</location>
    <log_format>eventchannel</log_format>
</localfile>
```

Restart agent:
```powershell
Restart-Service WazuhSvc
```

Verify in Wazuh Dashboard — look for `rule.groups: sysmon` in alerts.

---

## Phase 5 — Zeek + Wireshark Pipeline

> ⚠️ WSL2 cannot do live packet capture — virtual NIC only sees WSL2 traffic.
> Workflow: capture on Windows via Wireshark → export .pcap → analyze in Zeek on WSL2.

### 5.1 Install Wireshark (Windows)

Download from wireshark.org → install with **Npcap** (required for raw packet capture).

Capture on **Wi-Fi** interface → File → Save As → `.pcap`

Store captures in: `C:\Users\elli\Desktop\JOB\PHASE A\SOC ANALYST L1\pcap folder\`

### 5.2 Install Zeek (WSL2)
```bash
echo 'deb http://download.opensuse.org/repositories/security:/zeek/xUbuntu_22.04/ /' \
  | sudo tee /etc/apt/sources.list.d/security:zeek.list

curl -fsSL https://download.opensuse.org/repositories/security:zeek/xUbuntu_22.04/Release.key \
  | gpg --dearmor \
  | sudo tee /etc/apt/trusted.gpg.d/security_zeek.gpg > /dev/null

sudo apt update && sudo apt install zeek -y
echo 'export PATH=$PATH:/opt/zeek/bin' >> ~/.bashrc
source ~/.bashrc
zeek --version
```

### 5.3 Enable JSON Output
```bash
sudo nano /opt/zeek/share/zeek/site/local.zeek
# Add at bottom:
# @load policy/tuning/json-logs
```

### 5.4 Analyze a Capture
```bash
mkdir ~/zeek-analysis && cd ~/zeek-analysis

# Copy pcap from Windows
cp "/mnt/c/Users/elli/Desktop/JOB/PHASE A/SOC ANALYST L1/pcap folder/first-capture.pcap" .

# Run Zeek
zeek -r first-capture.pcap /opt/zeek/share/zeek/site/local.zeek
ls -la
# Produces: conn.log, dns.log, http.log, ssl.log, weird.log, x509.log, ntp.log
```

### 5.5 Pipe Zeek into OpenSearch
```bash
sudo tee /etc/logstash/conf.d/zeek.conf << 'EOF'
input {
  file {
    path => [
      "/home/ellianima/zeek-analysis/conn.log",
      "/home/ellianima/zeek-analysis/dns.log",
      "/home/ellianima/zeek-analysis/http.log",
      "/home/ellianima/zeek-analysis/ssl.log",
      "/home/ellianima/zeek-analysis/weird.log"
    ]
    codec => "json"
    type => "zeek"
    start_position => "beginning"
  }
}
output {
  if [type] == "zeek" {
    opensearch {
      hosts => ["https://172.23.201.127:9200"]
      index => "zeek-%{+YYYY.MM.dd}"
      user => "admin"
      password => "admin"
      ssl => true
      ssl_certificate_verification => false
    }
  }
}
EOF

# Fix permissions
sudo usermod -aG ellianima logstash
sudo rm /var/lib/logstash/plugins/inputs/file/.sincedb*
sudo systemctl restart logstash

# Verify
curl -k -u admin:admin "https://172.23.201.127:9200/zeek-*/_count"
# Should return count > 0
```

Create index pattern in Dashboard: `zeek-*` → time field: `@timestamp`

---

## Phase 6 — TheHive (On-Demand)

> ⚠️ TheHive runs on-demand only to preserve RAM.
> Start when needed, stop when done.

### 6.1 Install Docker
```bash
sudo apt install docker.io docker-compose -y
sudo systemctl start docker
sudo usermod -aG docker ellianima
newgrp docker
```

### 6.2 Deploy TheHive
```bash
mkdir ~/thehive && cd ~/thehive

cat > docker-compose.yml << 'EOF'
version: "3"
services:
  cassandra:
    image: cassandra:4
    environment:
      - CASSANDRA_CLUSTER_NAME=thehive
      - MAX_HEAP_SIZE=512M
      - HEAP_NEWSIZE=128M
    healthcheck:
      test: ["CMD", "cqlsh", "-e", "describe keyspaces"]
      interval: 30s
      timeout: 10s
      retries: 10

  thehive:
    image: strangebee/thehive:5.2
    depends_on:
      cassandra:
        condition: service_healthy
    ports:
      - "9000:9000"
    environment:
      - JVM_OPTS=-Xms512m -Xmx512m
      - TH_SECRET_KEY=MySecretKey123
      - CQL_HOSTNAMES=cassandra
EOF
```

### 6.3 Start / Stop TheHive
```bash
# Start
cd ~/thehive
docker-compose up -d

# Stop
docker-compose down
```

**URL:** `http://localhost:9000`
**Login:** `admin@thehive.local / secret`

> Cassandra takes ~2-3 minutes to initialize on first start.
> TheHive will be available ~1 minute after Cassandra shows healthy.

---

## Phase 7 — Kali Linux VM (Attacker) 🔜 Planned

VirtualBox VM settings:
- Name: Kali-Attacker
- RAM: 4096 MB
- Storage: 50 GB
- Network: Bridged Adapter

> ⚠️ NO Wazuh agent — pure attacker, zero SOC tools installed.

---

## Service Start Order
```bash
sudo systemctl start wazuh-indexer
sudo systemctl start wazuh-manager
sudo systemctl start wazuh-dashboard
sudo systemctl start logstash
# Zeek: run manually per analysis session
# TheHive: start on-demand via docker-compose
```

---

## Verification Checklist
[ ] WSL2 IP confirmed: hostname -I → 172.23.201.127
[ ] All 4 services active: systemctl status wazuh-*
[ ] Wazuh alert count > 0
[ ] Dashboard accessible: https://172.23.201.127
[ ] Agent (Candy-Cat-Silly-Fun) shows Active
[ ] FIM detecting file changes
[ ] MITRE ATT&CK panel showing data
[ ] Sysmon events visible (rule.groups: sysmon)
[ ] Zeek logs flowing: zeek-* index count > 0
[ ] TheHive accessible when started: http://localhost:9000

---

## Troubleshooting

| Problem | Cause | Fix |
|---|---|---|
| WSL2 IP changed after reboot | DHCP reassignment | Verify boot command in `/etc/wsl.conf` |
| Logstash crash loop | Missing ES plugin dependency | Reinstall `logstash-output-elasticsearch` |
| Filebeat pthread crash | WSL2 threading restriction | Use Logstash instead |
| Indexer security not initialized | First-time setup skipped | `sudo /usr/share/wazuh-indexer/bin/indexer-security-init.sh` |
| Dashboard "server not ready" | Wrong indexer IP | Set `opensearch.hosts` to WSL2 IP |
| MITRE field mapping errors | Wrong Logstash template | Apply Wazuh template via curl PUT |
| Alert count stays 0 | Logstash can't read alerts.json | `sudo usermod -aG wazuh logstash` |
| Logstash inactive after reboot | Started before Indexer ready | `ExecStartPre=/bin/sleep 30` in logstash.service |
| Zeek count stays 0 | Logstash can't read Zeek files | `sudo usermod -aG ellianima logstash` |
| TheHive exits immediately | Cassandra not ready yet | Use healthcheck in docker-compose — wait for healthy |
| Zeek bad checksum warnings | NIC hardware offloading | Add `-C` flag: `zeek -C -r capture.pcap` |

---

*Cerveaux Labs — Home SOC Operations*
*github.com/ellianima/home-soc-lab*
*Last updated: April 4, 2026*
