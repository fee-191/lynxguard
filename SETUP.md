# Hướng dẫn Setup Lab VMware

> Hướng dẫn từng bước dựng môi trường LynxGuard trên VMware Workstation. Toàn bộ hệ thống chạy ảo hóa — không cần phần cứng đặc biệt.

---

## Yêu cầu hệ thống

| Thành phần | Tối thiểu | Khuyến nghị |
|-----------|-----------|-------------|
| RAM | 16 GB | 32 GB |
| CPU | 4 cores | 8 cores |
| Disk | 150 GB | 250 GB |
| OS Host | Windows 10/11, Ubuntu 20.04+ | Ubuntu 22.04 |
| VMware | Workstation 16+ | Workstation 17 Pro |

---

## Tổng quan các VM

| VM | OS | RAM | vCPU | Disk | Role |
|----|-----|-----|------|------|------|
| pfsense-gw | pfSense 2.7 CE | 1 GB | 1 | 20 GB | Firewall / Gateway |
| web-server | Ubuntu 22.04 | 2 GB | 2 | 30 GB | Apache + ModSecurity |
| snort-ids | Ubuntu 22.04 | 2 GB | 2 | 25 GB | Snort 3 IDS |
| wazuh-siem | Ubuntu 22.04 | 4 GB | 4 | 50 GB | Wazuh Manager + Dashboard |
| zabbix-monitor | Ubuntu 22.04 | 2 GB | 2 | 30 GB | Zabbix Server |
| attacker-sim | Windows 10 | 4 GB | 2 | 40 GB | Testing / Attack simulation |

---

## Phần 1 — Cấu hình VMware Network

### 1.1 Tạo virtual networks

Trong VMware Workstation: **Edit → Virtual Network Editor**

| Network | Type | Subnet | Purpose |
|---------|------|--------|---------|
| VMnet0 | Bridged | (host network) | WAN — kết nối internet thật |
| VMnet1 | Host-only | 192.168.100.0/24 | LAN — mạng nội bộ |
| VMnet2 | Host-only | 192.168.200.0/24 | DMZ — web server |

### 1.2 Sơ đồ network

```
Internet (VMnet0 - Bridged)
        │
   ┌────┴────┐
   │ pfSense │  WAN: DHCP from host
   │  GW     │  LAN: 192.168.100.1
   │         │  DMZ: 192.168.200.1
   └─┬──────┬┘
     │      │
  VMnet1  VMnet2
  (LAN)   (DMZ)
     │      │
  ┌──┤   ┌──┤
  │  │   │  │
Wazuh  Snort  WebServer
Zabbix        (Apache+ModSec)
```

---

## Phần 2 — pfSense Firewall

### 2.1 Cài đặt

1. Download pfSense CE 2.7 ISO từ `pfsense.org`
2. Tạo VM mới: **2 NICs** (VMnet0 cho WAN, VMnet1 cho LAN)
3. Boot từ ISO → installer → **Quick/Easy Install**
4. Sau cài xong, thêm NIC thứ 3 (VMnet2) cho DMZ

### 2.2 Cấu hình cơ bản

Truy cập web GUI tại `192.168.100.1` (từ VM cùng VMnet1):

```
System > General Setup
  Hostname: pfsense-gw
  Domain: lab.local

Interfaces > WAN: DHCP (VMnet0)
Interfaces > LAN: Static 192.168.100.1/24
Interfaces > DMZ: Static 192.168.200.1/24
```

### 2.3 Firewall rules

**LAN → WAN (allow outbound):**
```
Action: Pass | Interface: LAN
Source: LAN net | Destination: any
Protocol: any
```

**WAN → DMZ (allow HTTP/HTTPS for testing):**
```
Action: Pass | Interface: WAN
Source: any | Destination: DMZ net
Protocol: TCP | Port: 80, 443
```

**DMZ → LAN (block by default — DMZ không được vào LAN):**
```
Action: Block | Interface: DMZ
Source: DMZ net | Destination: LAN net
```

### 2.4 Cấu hình syslog gửi về Wazuh

```
Status > System Logs > Settings
  Remote Logging: Enable
  Remote log servers: 192.168.100.10:514  (Wazuh IP)
  Remote Syslog Contents: Firewall Events, General
```

---

## Phần 3 — Web Server (Apache + ModSecurity)

### 3.1 Cài đặt Apache + ModSecurity

```bash
sudo apt update && sudo apt install -y apache2 libapache2-mod-security2

# Enable modules
sudo a2enmod security2 headers

# Cài OWASP Core Rule Set
sudo apt install -y modsecurity-crs
sudo ln -s /usr/share/modsecurity-crs /etc/apache2/modsecurity-crs

sudo systemctl restart apache2
```

### 3.2 Cấu hình ModSecurity

```bash
# /etc/apache2/mods-enabled/security2.conf
sudo nano /etc/modsecurity/modsecurity.conf

# Đổi SecRuleEngine từ DetectionOnly sang On
SecRuleEngine On
SecRequestBodyAccess On
SecResponseBodyAccess On
SecAuditEngine On
SecAuditLog /var/log/apache2/modsec_audit.log
```

### 3.3 Load OWASP CRS

```bash
# /etc/apache2/conf-available/modsecurity-crs.conf
sudo nano /etc/apache2/conf-available/modsecurity-crs.conf
```

```apache
Include /etc/modsecurity/modsecurity.conf
Include /usr/share/modsecurity-crs/crs-setup.conf
Include /usr/share/modsecurity-crs/rules/*.conf
```

```bash
sudo a2enconf modsecurity-crs
sudo systemctl restart apache2
```

---

## Phần 4 — Snort 3 IDS

### 4.1 Cài đặt Snort 3

```bash
# Dependencies
sudo apt install -y build-essential libpcap-dev libpcre3-dev \
  libdnet-dev zlib1g-dev liblzma-dev openssl libssl-dev \
  nghttp2 libnghttp2-dev luajit libluajit-5.1-dev

# Download và build Snort 3
wget https://github.com/snort3/snort3/archive/refs/tags/3.1.x.tar.gz
tar -xzf snort3-3.1.x.tar.gz && cd snort3-3.1.x
./configure_cmake.sh --prefix=/usr/local
cd build && make -j$(nproc) && sudo make install
```

### 4.2 Cấu hình Snort

```bash
# /usr/local/etc/snort/snort.lua
sudo nano /usr/local/etc/snort/snort.lua
```

```lua
-- Network configuration
HOME_NET = '192.168.100.0/24, 192.168.200.0/24'
EXTERNAL_NET = '!HOME_NET'

-- Alert output
alert_syslog =
{
    facility = 'daemon',
    level = 'alert',
    options = 'pid',
}
```

### 4.3 Chạy Snort ở chế độ IDS

```bash
# Monitor interface VMnet1 (LAN traffic)
sudo snort -c /usr/local/etc/snort/snort.lua \
  -i ens33 \
  --warn-all \
  -A alert_syslog \
  --daemonize
```

---

## Phần 5 — Wazuh SIEM

### 5.1 Cài đặt Wazuh (All-in-one)

```bash
# Download installer
curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh
curl -sO https://packages.wazuh.com/4.7/config.yml

# Sửa config.yml: set IP của server
nano config.yml
# nodes.indexer.ip: 192.168.100.10
# nodes.server.ip: 192.168.100.10
# nodes.dashboard.ip: 192.168.100.10

# Chạy installer
sudo bash wazuh-install.sh -a
```

### 5.2 Cài Wazuh Agent trên web server

```bash
# Trên web-server VM
curl -sO https://packages.wazuh.com/4.7/wazuh-agent-4.7.x_amd64.deb
sudo dpkg -i wazuh-agent-4.7.x_amd64.deb

# Sửa agent config để point về Wazuh Manager
sudo nano /var/ossec/etc/ossec.conf
# <server><address>192.168.100.10</address></server>

sudo systemctl start wazuh-agent
sudo systemctl enable wazuh-agent
```

### 5.3 Custom detection rules

Thêm vào `/var/ossec/etc/rules/local_rules.xml`:

```xml
<!-- Brute force: 5+ failed login trong 60 giây -->
<rule id="100200" level="10">
  <if_matched_sid>5501</if_matched_sid>
  <same_srcip />
  <timeframe>60</timeframe>
  <frequency>5</frequency>
  <description>Multiple authentication failures (brute force)</description>
</rule>

<!-- ModSecurity SQL injection alert -->
<rule id="100300" level="12">
  <decoded_as>apache-errorlog</decoded_as>
  <regex>ModSecurity.*SQL injection</regex>
  <description>SQL Injection attempt blocked by WAF</description>
</rule>

<!-- Snort alert forwarded -->
<rule id="100400" level="8">
  <decoded_as>snort</decoded_as>
  <description>Snort IDS alert detected</description>
</rule>
```

### 5.4 Truy cập Wazuh Dashboard

```
URL: https://192.168.100.10 (từ host machine)
Username: admin
Password: (generated during install, saved in wazuh-passwords.txt)
```

---

## Phần 6 — Zabbix Monitoring

### 6.1 Cài đặt Zabbix Server

```bash
# Zabbix 6.x trên Ubuntu 22.04
wget https://repo.zabbix.com/zabbix/6.4/ubuntu/pool/main/z/zabbix-release/zabbix-release_6.4-1+ubuntu22.04_all.deb
sudo dpkg -i zabbix-release_6.4-1+ubuntu22.04_all.deb
sudo apt update

sudo apt install -y zabbix-server-mysql zabbix-frontend-php \
  zabbix-apache-conf zabbix-sql-scripts zabbix-agent mysql-server

# Setup database
sudo mysql -e "CREATE DATABASE zabbix CHARACTER SET utf8mb4;"
sudo mysql -e "CREATE USER 'zabbix'@'localhost' IDENTIFIED BY 'zabbix_pass';"
sudo mysql -e "GRANT ALL ON zabbix.* TO 'zabbix'@'localhost';"
cat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | \
  zcat | sudo mysql zabbix

sudo systemctl start zabbix-server zabbix-agent apache2
sudo systemctl enable zabbix-server zabbix-agent apache2
```

### 6.2 Webhook Telegram

Trong Zabbix web UI:
```
Administration > Media types > Create media type
  Type: Webhook
  Name: Telegram
  Script: (xem file telegram_webhook.js trong repo)
  
Parameters:
  to: {ALERT.SENDTO}  (chat_id)
  subject: {ALERT.SUBJECT}
  message: {ALERT.MESSAGE}
  bot_token: YOUR_BOT_TOKEN
```

---

## Phần 7 — Telegram Bot

### 7.1 Tạo bot

1. Message `@BotFather` trên Telegram
2. `/newbot` → đặt tên → lấy **bot token**
3. `/start` bot → lấy **chat_id** từ `https://api.telegram.org/bot{TOKEN}/getUpdates`

### 7.2 Test thủ công

```bash
curl -s -X POST "https://api.telegram.org/bot{YOUR_TOKEN}/sendMessage" \
  -d chat_id="{YOUR_CHAT_ID}" \
  -d text="🔔 LynxGuard test alert — system online"
```

### 7.3 Python alert handler

```python
#!/usr/bin/env python3
import requests, json, sys

BOT_TOKEN = "YOUR_BOT_TOKEN"
CHAT_ID   = "YOUR_CHAT_ID"
THRESHOLD = 6   # Wazuh alert level threshold

def send_telegram(message: str) -> None:
    url = f"https://api.telegram.org/bot{BOT_TOKEN}/sendMessage"
    requests.post(url, data={"chat_id": CHAT_ID, "text": message, "parse_mode": "Markdown"})

def handle_wazuh_alert(alert: dict) -> None:
    level = alert.get("rule", {}).get("level", 0)
    if level < THRESHOLD:
        return
    source = alert.get("agent", {}).get("name", "unknown")
    rule   = alert.get("rule", {}).get("description", "")
    send_telegram(
        f"🚨 *Security Alert*\n"
        f"*Source:* `{source}`\n"
        f"*Rule:* {rule}\n"
        f"*Severity:* {level}/15"
    )

if __name__ == "__main__":
    alert = json.load(sys.stdin)
    handle_wazuh_alert(alert)
```

---

## Phần 8 — Verify Setup

```bash
# Test ModSecurity WAF
curl -X GET "http://192.168.200.X/test?id=1' OR '1'='1"
# Expected: 403 Forbidden, ModSecurity log entry

# Test Snort detection
nmap -sS 192.168.200.X  # từ attacker VM
# Expected: Snort alert trong /var/log/syslog

# Test Wazuh correlation
# Gửi 6 failed SSH logins liên tiếp → Wazuh rule 100200 trigger

# Verify Telegram bot
# Sau khi trigger bất kỳ alert → check Telegram group nhận được message
```

---

*Môi trường này được dựng cho mục đích học tập và nghiên cứu bảo mật. Không sử dụng trong production.*
