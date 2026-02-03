# SOC Lab Monitoring: Prometheus + Grafana + OPNsense Suricata → Wazuh

This repo documents a **working, repeatable build** of:

- **Prometheus + Grafana** (Docker) monitoring an **Ubuntu Server** (Node Exporter)
- **OPNsense VM** running **Suricata in IDS mode**
- **Wazuh Manager (Ubuntu)** ingesting Suricata **EVE JSON** via **Wazuh Agent on OPNsense**
- Viewing Suricata detections in the Wazuh UI (**Threat Hunting** / **Events**)

> Goal: a portfolio-grade SOC homelab pipeline that produces **real IDS alerts** from Kali scans and shows them in **Wazuh** while monitoring the Ubuntu host with **Prometheus/Grafana**.

---

## Architecture

### Network zones (recommended)
A 3-zone layout avoids `$HOME_NET` suppression and makes IDS alerts consistent.

- **WAN** (external)
- **LAN** (protected assets) — Ubuntu Wazuh, Windows Server, etc.
- **ATTACK_NET / OPT1** (attacker segment) — Kali

```
Kali (ATTACK_NET) ──> OPNsense ──> LAN assets
                        │
                        └─ Suricata IDS enabled on LAN interface
```

### Components
- Ubuntu Server (Wazuh Manager + Indexer + Dashboard) — `192.168.1.20`
- OPNsense VM — LAN gateway - `192.168.1.1`
- Kali VM on ATTACK_NET (OPT1) - `192.168.4.0/24`
- Prometheus + Grafana running on Ubuntu (Docker)
- Node Exporter on Ubuntu for system metrics

---

## Prerequisites

- Ubuntu Server on LAN with static IP - `192.168.1.20`
  ![5b edit yaml file static ip ubuntu](https://github.com/user-attachments/assets/825db484-136d-48c8-83cb-f337c6c359b9)

- OPNsense VM routing between ATTACK_NET and LAN
- Kali on ATTACK_NET able to reach LAN via firewall rules
- Docker installed on Ubuntu (for Prometheus/Grafana)
- Internet access from Ubuntu to pull container images and IDS rules

---

# Part A — Prometheus + Grafana (Docker) for Ubuntu monitoring

## A1) Install Node Exporter (Ubuntu)
```bash
sudo apt update
sudo apt install -y prometheus-node-exporter
sudo systemctl enable --now prometheus-node-exporter
curl -s http://localhost:9100/metrics | head
```

## A2) Deploy Prometheus + Grafana via Docker
Create and configure directories:

```bash
sudo mkdir -p /opt/monitoring/{prometheus,grafana}
sudo mkdir -p /opt/monitoring/prometheus/data
```

Set Ownership & Permissions:

#configure directories
```bash
sudo chown -R root:root /opt/monitoring/prometheus
sudo chmod 755 /opt/monitoring/prometheus
```

#configure Prometheus data directory (container-writable):
```bash
sudo chown -R 65534:65534 /opt/monitoring/prometheus/data
```

#configure grafana
```bash
sudo chown -R 472:472 /opt/monitoring/grafana
```

Create Immutable Prometheus Config:

```bash
sudo nano /opt/monitoring/prometheus/prometheus.yml

global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: "node_exporter"
    static_configs:
      - targets: ["host.docker.internal:9100"]

```
Lock it:
```bash
sudo chmod 644 /opt/monitoring/prometheus/prometheus.yml
```
Run Prometheus (Hardened Container):
```bash

docker run -d \
  --name prometheus \
  --restart unless-stopped \
  --read-only \
  --cap-drop ALL \
  --network host \
  -v /opt/monitoring/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro \
  -v /opt/monitoring/prometheus/data:/prometheus \
  prom/prometheus


```

Run Grafana
```bash
docker run -d \
  --name grafana \
  --restart unless-stopped \
  -p 3000:3000 \
  -v /opt/monitoring/grafana:/var/lib/grafana \
  grafana/grafana-oss
```

Harden Prometheus:
 - Bind Prometheus to LAN only:
```bash
iptables -A INPUT -p tcp --dport 9090 -s 192.168.1.0/24 -j ACCEPT
iptables -A INPUT -p tcp --dport 9090 -j DROP
```


### Access
- Prometheus: `http://<ubuntu-ip>:9090`
- Grafana: `http://<ubuntu-ip>:3000` (default login `admin/admin`)

### NB:
 - Prometheus and Grafana were deployed via Docker following upstream recommendations.
 -   root:root → config integrity
 -   UID 65534 → Prometheus runtime user
 -   UID 472 → Grafana runtime user
 - Configuration files are mounted read-only and owned by root to prevent monitoring tampering.
 - Node Exporter runs on the host to ensure metrics availability even if containers fail.


## A3) Grafana: add Prometheus data source + import dashboard
1. Grafana → **Connections** → **Data sources** → **Prometheus**
 ![7m add data source to grafana](https://github.com/user-attachments/assets/3fd1f7ac-dc32-477d-9f0d-0bda10a77cc5)
  
   
3. URL: `http://prometheus:9090`
4. Import Node Exporter dashboard:
   - “Node Exporter Full” (Grafana Labs dashboard)
  ![7n 9 grafana dashboard configured with node exporter full](https://github.com/user-attachments/assets/21b7ece9-70e1-4193-8378-e09d4fbcd9a2)


## A4) Grafana Alerts (examples)
Create alert rules for:
- CPU usage high
- Memory pressure
- Disk filling up
- Node down (exporter unreachable)
![7p configured alerts on Grafana](https://github.com/user-attachments/assets/dba440e9-0f83-4949-b8a3-1af9f610e78c)

---

# Part B — OPNsense Suricata IDS + forwarding to Wazuh

## B1) Enable Suricata IDS on OPNsense
OPNsense Web UI:
1. **Services → Intrusion Detection**
2. Enable Intrusion Detection
3. Select interface(s): **LAN,OPT1,WAN**
4. Start in **IDS mode** first (IPS later if supported)
5. Enable EVE JSON logging (`eve.json`)
![11b configure IDS](https://github.com/user-attachments/assets/ccad0dfb-497d-4e21-a922-4f2207d3a5fe)

### Rule feeds
Enable **Emerging Threats Open** - for exploits, scan, recon, malware. Avoid turning on “everything” blindly; start small, tune as you progress..
![11c enable rules](https://github.com/user-attachments/assets/f25cad7d-20a5-4eb7-86d6-101761ed73a7)

## B2) Make Kali appear "external" to LAN rules
Use ATTACK_NET (OPT1) so scans are not treated as $HOME_NET traffic.

### Example firewall rules
- On OPT1: allow `IN OPT1 net → LAN net` (any/needed protocols)
- On LAN: allow return traffic if needed (stateful rules usually suffice; validate with live firewall logs)

## B3) Validate Suricata detects Kali scans
From Kali:
```bash
nmap -A 192.168.1.0/24
```
OPNsense → Intrusion Detection → Alerts should show hits.

---

# Part C — Ingest Suricata EVE JSON into Wazuh

## C1) Install Wazuh Agent on OPNsense
Install from the System --> Firmware --> Plugins (check "show community plugins")
![11d enable wazuh-agent opnsense](https://github.com/user-attachments/assets/55431500-48c8-493a-9976-271adb025f2b)


```
Configure `/var/ossec/etc/ossec.conf`  in OPNSense shell to:
1) connect to manager, and
2) read Suricata EVE JSON from OPNsense:


<-- Minimal ossec.conf configuration -->
<ossec_config>
  <client>
    <server>
      <address>192.168.1.20</address>
      <port>1514</port>
      <protocol>tcp</protocol>
    </server>
  </client>

  <localfile>
    <log_format>json</log_format>
    <location>/var/log/suricata/eve.json</location>
  </localfile>
</ossec_config>


See `wazuh/opnsense-ossec.conf.example`.

Restart agent:
```sh
service wazuh-agent restart
```

Confirm on Wazuh manager:
```bash
/var/ossec/bin/agent_control -lc
```
![13 confirm opnsense agent wazuh manager](https://github.com/user-attachments/assets/f60cf382-8697-4306-a77f-785236d6c6ca)

## C2) (Optional) Enable archives.json
If you want raw archives at:
`/var/ossec/logs/archives/archives.json`

Enable in `/var/ossec/etc/ossec.conf`:
```xml
<logall>yes</logall>
<logall_json>yes</logall_json>
```
Restart:
```bash
sudo systemctl restart wazuh-manager
```

## C3) Confirm alerts from OPNSense reach Wazuh Manager:

On Kali, run:
```bash
nmap -A 192.168.1.20/24
```
On the Wazuh Manager, run:
```bash
sudo tail -f /var/ossec/logs/alerts/alerts.json
```
![14 confirm alerts in wazuh manager ](https://github.com/user-attachments/assets/69ba944b-bd16-477c-b229-36c21bede35c)

## C4) (Optional) Add Suricata elevation rules (Wazuh)
If you want a custom rule file, and not the one from the rule feeds, you can create one:
`/var/ossec/etc/rules/suricata_rules.xml`

Always validate rules before restarting:
```bash
sudo /var/ossec/bin/wazuh-analysisd -t
sudo systemctl restart wazuh-manager
```

## C5) View Suricata in Wazuh UI
Depending on Wazuh UI version/layout, Suricata alerts may appear under:
- **Threat Hunting**
- **Events**
- **Discover** (data view: `wazuh-alerts-*`)

> In our final state, Suricata appeared under **Threat Hunting** and **Events**.
![11i confirm suricata in wazuh](https://github.com/user-attachments/assets/d41ec482-0ddc-4c5e-bdfe-d55f28c7c26b)

---

# Troubleshooting & Lessons Learned:

- Windows 11 **Insider Dev** build broke browser access on host machine to lab services (while Kali VM could access them), had to repair Windows to fix it.
- Prometheus recommended Docker; `prometheus.yml` “missing” was expected until mounted into container.
- Avoid custom decoders unless absolutely necessary; one bad XML can stop Wazuh from starting cleanly.
- OPNsense OPT interfaces can advertise IPv6 RA by default; DHCPv4 failures can be caused by RA preference.
- Wazuh archives may not exist unless enabled; alerts can work perfectly without archives.
