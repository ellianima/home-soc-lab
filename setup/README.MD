# SOC Lab Setup Guide

Full reproducible setup documentation for the Home SOC Operations Lab.
Environment: WSL2 Ubuntu 22.04 on Windows 11 host, Kali Linux VM (attacker).

---

## Prerequisites

- Windows 11 (host machine)
- WSL2 enabled with Ubuntu 22.04 LTS
- Oracle VirtualBox 7.x (for Kali attacker VM)
- Kali Linux ISO (Rolling) — [kali.org/get-kali](https://www.kali.org/get-kali/)
- Minimum host RAM: 12GB (WSL2: 8GB allocated, Kali VM: 4GB)

---

## Architecture Overview
```
Windows 11 (Host — 12GB RAM)
├── WSL2 Ubuntu 22.04 (8GB allocated via .wslconfig)
│   ├── Wazuh Manager 4.14.4
│   ├── Wazuh Indexer 4.14.4 (OpenSearch 2.19.4)
│   ├── Wazuh Dashboard 4.14.4
│   └── Logstash 7.17.29 (alert pipeline)
├── Wazuh Agent 4.14.4 (Windows self-monitoring)
└── VirtualBox
└── Kali Linux VM — 4GB RAM [🔜 Planned — attacker]
```
---

## Phase 1 — WSL2 Setup

### 1.1 Install WSL2 (no distro)

```powershell
# Run as Administrator in PowerShell
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
# Set password (nothing shows when typing — this is normal)
```

### 1.5 Configure WSL2 Memory Limit

Create `C:\Users\elli\.wslconfig`:

```ini
[wsl2]
memory=8GB
swap=2GB

[boot]
command = sysctl -w kernel.perf_event_paranoid=1
```

Verify after restarting WSL2:

```bash
free -h
# Should show ~7.8GB total
```

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
sudo systemctl status wazuh-manager
```

### 2.4 Generate SSL Certificates

```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-certs-tool.sh
curl -sO https://packages.wazuh.com/4.14/config.yml
```

Edit `config.yml` — replace all IP placeholders with WSL2 IP:

```bash
hostname -I
# Use this IP (e.g. 172.23.201.127) for all three nodes below
nano config.yml
```

```yaml
nodes:
  indexer:
    - name: node-1
      ip: "172.23.201.127"
  server:
    - name: wazuh-1
      ip: "172.23.201.127"
  dashboard:
    - name: dashboard
      ip: "172.23.201.127"
```

```bash
sudo bash wazuh-certs-tool.sh -A
```

### 2.5 Install Wazuh Indexer

```bash
sudo apt-get install wazuh-indexer -y

# Copy certificates
sudo mkdir /etc/wazuh-indexer/certs
sudo cp ~/wazuh-certificates/node-1.pem /etc/wazuh-indexer/certs/indexer.pem
sudo cp ~/wazuh-certificates/node-1-key.pem /etc/wazuh-indexer/certs/indexer-key.pem
sudo cp ~/wazuh-certificates/root-ca.pem /etc/wazuh-indexer/certs/root-ca.pem
sudo cp ~/wazuh-certificates/admin.pem /etc/wazuh-indexer/certs/admin.pem
sudo cp ~/wazuh-certificates/admin-key.pem /etc/wazuh-indexer/certs/admin-key.pem
sudo bash -c 'chmod 400 /etc/wazuh-indexer/certs/*'
sudo chown -R wazuh-indexer:wazuh-indexer /etc/wazuh-indexer/certs

# Start indexer
sudo systemctl daemon-reload
sudo systemctl enable wazuh-indexer
sudo systemctl start wazuh-indexer

# Initialize security plugin
sudo /usr/share/wazuh-indexer/bin/indexer-security-init.sh
```

### 2.6 Install Wazuh Dashboard

```bash
sudo apt-get install wazuh-dashboard -y

# Copy certificates
sudo mkdir /etc/wazuh-dashboard/certs
sudo cp ~/wazuh-certificates/dashboard.pem /etc/wazuh-dashboard/certs/dashboard.pem
sudo cp ~/wazuh-certificates/dashboard-key.pem /etc/wazuh-dashboard/certs/dashboard-key.pem
sudo cp ~/wazuh-certificates/root-ca.pem /etc/wazuh-dashboard/certs/root-ca.pem
sudo bash -c 'chmod 400 /etc/wazuh-dashboard/certs/*'
sudo chown -R wazuh-dashboard:wazuh-dashboard /etc/wazuh-dashboard/certs

# Fix dashboard config — point to indexer IP (not localhost)
sudo nano /etc/wazuh-dashboard/opensearch_dashboards.yml
# Change: opensearch.hosts: https://172.23.201.127:9200

# Start dashboard
sudo systemctl daemon-reload
sudo systemctl enable wazuh-dashboard
sudo systemctl start wazuh-dashboard
```

**Dashboard:** `https://172.23.201.127` or `https://localhost`
**Login:** `admin / admin`

---

## Phase 3 — Logstash Alert Pipeline

> ⚠️ Filebeat is NOT used — it crashes on WSL2 due to pthread threading restrictions.
> Logstash with the OpenSearch plugin is the replacement.

### 3.1 Add Elastic Repository

```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | \
  sudo gpg --dearmor -o /usr/share/keyrings/elastic-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/elastic-keyring.gpg] \
  https://artifacts.elastic.co/packages/7.x/apt stable main" | \
  sudo tee /etc/apt/sources.list.d/elastic-7.x.list

sudo apt-get update
sudo apt-get install logstash -y
```

### 3.2 Install OpenSearch Output Plugin

```bash
# Install OpenSearch plugin (required — Wazuh Indexer is OpenSearch, not Elasticsearch)
sudo /usr/share/logstash/bin/logstash-plugin install logstash-output-opensearch

# Keep Elasticsearch plugin installed (internal dependency)
# Do NOT remove logstash-output-elasticsearch
```

### 3.3 Configure Logstash Pipeline

```bash
# Download Wazuh index template
sudo curl -so /etc/filebeat/wazuh-template.json \
  https://raw.githubusercontent.com/wazuh/wazuh/v4.14.4/extensions/elasticsearch/7.x/wazuh-template.json

# Apply template to indexer
curl -k -u admin:admin -X PUT "https://172.23.201.127:9200/_template/wazuh" \
  -H 'Content-Type: application/json' \
  -d @/etc/filebeat/wazuh-template.json

# Create pipeline config
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

# Add logstash user to wazuh group (file permission fix)
sudo usermod -aG wazuh logstash

# Store indexer credentials
sudo /var/ossec/bin/wazuh-keystore -f indexer -k username -v admin
sudo /var/ossec/bin/wazuh-keystore -f indexer -k password -v admin

# Enable and start
sudo systemctl enable logstash
sudo systemctl start logstash
```

### 3.4 Verify Pipeline

```bash
# Wait ~2 minutes for Logstash to initialize, then:
curl -k -u admin:admin "https://172.23.201.127:9200/wazuh-alerts-*/_count"
# Should return count > 0
```

---

## Phase 4 — Windows Agent

### 4.1 Deploy via Wazuh Dashboard

Dashboard → Server Management → Endpoints Summary → Deploy new agent
OS: Windows
Architecture: x86_64
Server address: 172.23.201.127

Run the generated PowerShell command as Administrator on Windows.

### 4.2 Configure FIM

Edit `C:\Program Files (x86)\ossec-agent\ossec.conf` as Administrator.
Add inside `<syscheck>` section:

```xml
<directories realtime="yes" report_changes="yes">C:\Users\Public</directories>
<directories realtime="yes">C:\Windows\System32</directories>
<directories realtime="yes">C:\ProgramData</directories>
```

Restart agent:

```powershell
NET STOP WazuhSvc
NET START WazuhSvc
```

---

## Phase 5 — Zeek 🔜 Planned

```bash
# Add Zeek repository
echo 'deb http://download.opensuse.org/repositories/security:/zeek/xUbuntu_22.04/ /' \
  | sudo tee /etc/apt/sources.list.d/security:zeek.list

curl -fsSL https://download.opensuse.org/repositories/security:zeek/xUbuntu_22.04/Release.key \
  | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/security_zeek.gpg > /dev/null

sudo apt update && sudo apt install zeek -y

# Configure interface in /opt/zeek/etc/node.cfg
# Deploy
sudo /opt/zeek/bin/zeekctl deploy
```

---

## Phase 6 — TheHive 🔜 Planned

```bash
# Install Docker
sudo apt install docker.io docker-compose -y
sudo systemctl enable docker
sudo systemctl start docker

# Deploy TheHive stack
mkdir ~/thehive && cd ~/thehive
# (docker-compose.yml TBD)
docker-compose up -d
# URL: http://172.23.201.127:9000
# Login: admin@thehive.local / secret
```

---

## Phase 7 — Kali Linux VM (Attacker) 🔜 Planned

VirtualBox VM settings:
Name: Kali-Attacker
RAM: 4096 MB
Storage: 50 GB
Network: Bridged Adapter
Notes: NO Wazuh agent — pure attacker, zero SOC tools

---

## Service Start Order

Always start in this order:

```bash
sudo systemctl start wazuh-indexer
sudo systemctl start wazuh-manager
sudo systemctl start wazuh-dashboard
sudo systemctl start logstash
```

---

## Verification Checklist

[ ] WSL2 IP confirmed (hostname -I)
[ ] All 4 services active (systemctl status)
[ ] Alert count > 0 in indexer
[ ] Dashboard accessible (https://172.23.201.127)
[ ] Agent 001 (Candy-Cat-Silly-Fun) shows Active
[ ] FIM detecting file changes
[ ] MITRE ATT&CK panel showing data
[ ] Logstash running without crash loops

---

## Troubleshooting

| Problem                            | Cause                                                    | Fix                                                          |
| ---------------------------------- | -------------------------------------------------------- | ------------------------------------------------------------ |
| WSL2 IP changed after reboot       | DHCP reassignment                                        | `hostname -I` to get new IP, update configs                  |
| Logstash crash loop                | Missing Elasticsearch plugin dependency                  | Reinstall `logstash-output-elasticsearch`                    |
| Filebeat pthread crash             | WSL2 kernel threading restriction                        | Don't use Filebeat — use Logstash instead                    |
| Indexer security not initialized   | First-time setup step skipped                            | `sudo /usr/share/wazuh-indexer/bin/indexer-security-init.sh` |
| Dashboard shows "server not ready" | Dashboard config pointing to wrong indexer IP            | Set `opensearch.hosts` to WSL2 IP, not localhost             |
| MITRE field mapping errors         | Logstash default template used instead of Wazuh template | Apply Wazuh template via curl PUT                            |
| Alert count stays 0                | Logstash can't read alerts.json                          | `sudo usermod -aG wazuh logstash`                            |

---

_Cerveaux Labs — Home SOC Operations_
_github.com/ellianima/home-soc-lab_
