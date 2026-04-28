# Suricata — Docker Test Environment

Suricata container pre-configured to sniff the host's main network interface, generate quick alerts, and forward them to Wazuh via `eve.json`. Designed to validate the detection pipeline locally.

## Stack

| Component | Technology |
|-----------|------------|
| IDS | Suricata 8.x (`jasonish/suricata:latest`) |
| Config | `suricata.yaml` (based on `suricata-merged.yaml`) |
| Rules | `rules/local.rules` (SID 9000001–9000099) |
| Wazuh rules | `rules/wazuh_local_rules.xml` |
| Wazuh output | `logs/eve.json` |
| Orchestration | Docker Compose |

## Structure

```
suricata prueba/
├── docker-compose.yml         ← container orchestration
├── suricata.yaml              ← Suricata config adapted for Docker
├── .env                       ← network interface (SURICATA_IFACE)
├── rules/
│   ├── local.rules            ← test detection rules (easy to trigger)
│   └── wazuh_local_rules.xml  ← custom Wazuh rules for local SIDs
└── logs/                      ← eve.json · fast.log · suricata.log
```

## Architecture

```
Host NIC (eth0 / ens33 / ...)
        ↓  network_mode: host
  Suricata (Docker)
        ↓
  logs/eve.json  ←──  Wazuh agent reads this file
  logs/fast.log  ←──  plain-text alerts (quick check)
```

## Setup

### 1. Configure the interface

Edit `.env` with the interface you want to monitor:

```bash
# Linux — find your main interface:
ip route | grep default | awk '{print $5}'

# Examples: eth0, ens33, enp3s0, wlan0
SURICATA_IFACE=eth0
```

> **Windows (Docker Desktop / WSL2):** inside the container the interface is always `eth0` (WSL2 virtual interface). Suricata sees Docker and WSL2 traffic, not the physical Windows NIC.

### 2. Start

```bash
docker-compose up -d
```

### 3. Verify it started

```bash
docker-compose logs -f
# Expected: "All AFP capture threads are running"
```

---

## Viewing alerts

### Plain text — fastest

```bash
tail -f logs/fast.log
```

Output format:
```
04/28/2026-17:32:11.123  [**] [1:9000001:1] [LOCAL] ICMP Echo Request (ping) [**] ...
```

### JSON for Wazuh

```bash
# Alerts only
tail -f logs/eve.json | grep '"event_type":"alert"'

# Pretty print
tail -f logs/eve.json | grep '"event_type":"alert"' | python -m json.tool
```

### Container stats

```bash
docker exec suricata-wazuh suricatasc -c /var/run/suricata/suricata-command.socket dump-counters
```

---

## Triggering test alerts

### ICMP — SID 9000001/9000002
```bash
ping -c 3 8.8.8.8
```

### HTTP test server — SID 9000010–9000013
```bash
# Normal request
curl -i http://localhost:8080/

# 4xx errors
curl -i http://localhost:8080/401
curl -i http://localhost:8080/403
curl -i http://localhost:8080/404

# 5xx errors
curl -i http://localhost:8080/503

# Rate limiting
curl -i http://localhost:8080/429
```

### Tool user agents — SID 9000020–9000024
```bash
# curl already triggers SID 9000020
curl http://localhost:8080/

# wget
wget -q -O /dev/null http://localhost:8080/

# python-requests
python3 -c "import requests; requests.get('http://localhost:8080/')"
```

### URI injection — SID 9000030–9000033
```bash
# SQL injection
curl "http://localhost:8080/?id=1%27%20OR%20%271%27%3D%271"

# Path traversal
curl "http://localhost:8080/../../../etc/passwd"

# XSS
curl "http://localhost:8080/?q=<script>alert(1)</script>"
```

### Service connections — SID 9000040–9000047
```bash
# SSH (triggers even if no server is listening)
ssh -o ConnectTimeout=2 localhost 2>/dev/null || true

# RDP / SMB via nmap (SYN only, no service needed)
nmap -p 3389,445 localhost
```

### Port scan — SID 9000050/9000051
```bash
# SYN scan (requires nmap)
nmap -sS localhost

# Fast range scan
nmap -p 1-1000 localhost
```

### Brute force simulation — SID 9000080/9000081
```bash
# 10 requests returning 401 (triggers after 5 in 60s)
for i in {1..10}; do curl -s http://localhost:8080/401 > /dev/null; done

# 10 requests returning 403
for i in {1..10}; do curl -s http://localhost:8080/403 > /dev/null; done
```

---

## Local rules reference

| SID | Category | Detects | How to trigger |
|-----|----------|---------|----------------|
| 9000001 | ICMP | Echo Request (ping) | `ping 8.8.8.8` |
| 9000002 | ICMP | Echo Reply | `ping 8.8.8.8` |
| 9000010 | HTTP | Request to port 8080 | `curl localhost:8080` |
| 9000011 | HTTP | 4xx response from test server | `curl localhost:8080/404` |
| 9000012 | HTTP | 5xx response from test server | `curl localhost:8080/500` |
| 9000013 | HTTP | 429 response from test server | `curl localhost:8080/429` |
| 9000020 | HTTP | curl user-agent | `curl localhost:8080` |
| 9000021 | HTTP | wget user-agent | `wget localhost:8080` |
| 9000022 | HTTP | python-requests user-agent | `python3 -c "import requests..."` |
| 9000023 | HTTP | Nmap HTTP scan | `nmap --script http-title localhost` |
| 9000024 | HTTP | Nikto scanner | `nikto -h localhost:8080` |
| 9000030 | Web Attack | SQL injection in URI | `curl "...?id=1' OR '1'='1"` |
| 9000031 | Web Attack | Path traversal `../` | `curl ".../../../etc/passwd"` |
| 9000032 | Web Attack | Path traversal URL-encoded | `curl "...%2e%2e%2f"` |
| 9000033 | Web Attack | XSS in URI | `curl "...?q=<script>"` |
| 9000040 | TCP | SSH connection attempt | `ssh localhost` |
| 9000041 | TCP | Telnet (cleartext) | `telnet localhost` |
| 9000042 | TCP | RDP attempt | `nmap -p 3389 localhost` |
| 9000043 | TCP | SMB attempt | `nmap -p 445 localhost` |
| 9000044 | TCP | MySQL attempt | `nmap -p 3306 localhost` |
| 9000045 | TCP | PostgreSQL attempt | `nmap -p 5432 localhost` |
| 9000046 | TCP | Redis (unauthenticated?) | `nmap -p 6379 localhost` |
| 9000047 | TCP | MongoDB attempt | `nmap -p 27017 localhost` |
| 9000050 | Scan | Port scan (15+ SYN in 10s) | `nmap -sS localhost` |
| 9000051 | Scan | UDP scan (10+ UDP in 10s) | `nmap -sU localhost` |
| 9000061 | DNS | Long subdomain — DGA/C2 pattern | long subdomain DNS query |
| 9000070 | TLS | TLS on non-standard port | TLS connection to port != 443 |
| 9000080 | Brute Force | 5+ HTTP 401 in 60s | `for i in {1..10}; do curl .../401; done` |
| 9000081 | Brute Force | 5+ HTTP 403 in 60s | `for i in {1..10}; do curl .../403; done` |

---

## Wazuh integration

### 1. Configure the agent to read eve.json

Add to `/var/ossec/etc/ossec.conf` on the host running the Wazuh agent:

```xml
<localfile>
  <log_format>json</log_format>
  <location>/absolute/path/to/suricata prueba/logs/eve.json</location>
</localfile>
```

Windows agent path:
```xml
<location>C:\Users\Alejandro\Desktop\Desarrollo\Herramientas\suricata prueba\logs\eve.json</location>
```

Restart the agent:
```bash
# Linux
sudo systemctl restart wazuh-agent

# Windows PowerShell (admin)
Restart-Service WazuhSvc
```

### 2. Load the custom Wazuh rules

Copy `rules/wazuh_local_rules.xml` to the **Wazuh Manager**:

```bash
cp rules/wazuh_local_rules.xml /var/ossec/etc/rules/local_suricata_rules.xml
```

Restart the manager:
```bash
systemctl restart wazuh-manager
# or
/var/ossec/bin/wazuh-control restart
```

### 3. Wazuh rule hierarchy

```
Built-in rule (group: suricata)
  └─ 100000 level 3  → any [LOCAL] signature
       ├─ 100001/100002  level 3  → ICMP ping/reply
       ├─ 100010         level 3  → HTTP request to :8080
       ├─ 100011         level 6  → HTTP 4xx/5xx/429 errors
       ├─ 100020         level 4  → automation tools (curl/wget/python)
       ├─ 100023         level 8  → security scanners (nmap/nikto) [T1595]
       ├─ 100030         level 10 → web attacks [T1190]
       │    ├─ 100031    level 12 → SQL injection
       │    └─ 100033    level 10 → XSS
       ├─ 100040         level 5  → SSH [T1021.004]
       ├─ 100041         level 9  → Telnet cleartext [T1021]
       ├─ 100042         level 6  → RDP [T1021.001]
       ├─ 100043         level 6  → SMB [T1021.002]
       ├─ 100044         level 6  → databases (MySQL/PG/Redis/Mongo)
       ├─ 100050/100051  level 10 → port scan [T1046]
       ├─ 100061         level 10 → DNS DGA/C2 [T1568.002]
       ├─ 100070         level 10 → TLS non-standard port [T1571]
       ├─ 100080         level 12 → HTTP brute force 401 [T1110]
       └─ 100081         level 11 → resource enumeration 403 [T1595.003]
```

---

## Useful commands

```bash
# Stop
docker-compose down

# Rebuild after changes to suricata.yaml or local.rules
docker-compose down && docker-compose up -d

# Validate config without starting the container
docker run --rm \
  -v "$(pwd)/suricata.yaml:/etc/suricata/suricata.yaml:ro" \
  -v "$(pwd)/rules:/etc/suricata/rules:ro" \
  jasonish/suricata:latest -T -c /etc/suricata/suricata.yaml

# Watch Suricata internal stats in real time
watch -n2 'docker exec suricata-wazuh cat /var/log/suricata/stats.log | tail -20'

# Clear logs for a clean test run
> logs/eve.json && > logs/fast.log
```

---

## Differences vs production config (`Suricata/suricata-merged.yaml`)

| Parameter | This environment (test) | Production |
|-----------|------------------------|------------|
| `HOME_NET` | All RFC1918 private ranges | Specific environment subnets |
| `checksum-validation` | `no` — virtual NICs skip offloading | `yes` |
| `fast.log` | Enabled — useful in terminal | Disabled (redundant with eve-log) |
| `sgh-mpm-caching` | Disabled | Enabled (faster restarts) |
| `detect.profile` | `medium` | `high` |
| Rules | `local.rules` only | `suricata.rules` (ET community + local) |
| Interfaces | Single (`-i eth0`) | Dual: SPAN + management |

## Windows Docker Desktop limitation

`network_mode: host` maps to WSL2's virtual network, **not** the physical Windows NIC.

| Traffic | Visible |
|---------|---------|
| Docker containers | ✅ |
| Test server (port 8080) | ✅ |
| Generated from WSL2 | ✅ |
| Windows host (browser, native apps) | ❌ |

For full Windows host monitoring → install Suricata natively on Windows or configure a SPAN port on the switch/router.
