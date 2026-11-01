# Axion Infrastructure Monitoring

**Step-by-step Azure implementation guide**

Stack: Prometheus · Grafana · Node Exporter  |  OS: Ubuntu 24.04 LTS  |  Environment: Azure Virtual Machines

## 1. Overview

This guide sets up monitoring for the Linux infrastructure that runs the Axion application. **Prometheus** collects server metrics, **Node Exporter** exposes them from each VM, and **Grafana** displays them in dashboards.

### Architecture

| VM | Private IP | Components |
| --- | --- | --- |
| Frontend VM | `10.0.2.4` | React/Nginx, Prometheus, Grafana, Node Exporter |
| Backend VM | `10.0.4.4` | FastAPI, Node Exporter |

<img width="678" height="318" alt="image" src="https://github.com/user-attachments/assets/baad83c9-954a-4f70-8bd6-d7bb71a141d4" />


Prometheus on the frontend VM scrapes both exporters every 15 seconds, and Grafana only ever talks to Prometheus.

### How it works

1. Node Exporter runs on each VM and exposes Linux system metrics (CPU, memory, disk, network).
2. Prometheus scrapes both exporters every 15 seconds: the frontend one locally, the backend one over the private VNet.
3. Grafana queries Prometheus and presents the metrics in dashboards.

### Implementation roadmap

| Step | Task | VM |
| --- | --- | --- |
| 1 | Open the required network rules (NSG) | Azure Portal |
| 2 | Install Node Exporter | Frontend |
| 3 | Install and configure Prometheus | Frontend |
| 4 | Install Node Exporter | Backend |
| 5 | Test private connectivity | Frontend → Backend |
| 6 | Install Grafana and add the data source | Frontend |
| 7 | Verify targets and import a dashboard | Grafana / Prometheus |

## 2. Prerequisites and Network Security

Before you start, confirm that both VMs are running, you have `sudo` access on each, and the VMs can reach each other over the Azure Virtual Network.

### Required ports

| Port | Service | Who needs access |
| --- | --- | --- |
| `3000` | Grafana | Administrator's trusted IP or VPN |
| `9090` | Prometheus UI | Administrator's trusted IP or VPN (only if direct access is needed) |
| `9100` | Node Exporter | Monitoring traffic only. **Never public.** |
| `8000` | FastAPI | As required by the application |

## 3. Frontend VM: Install Node Exporter

**Objective:** Collect CPU, memory, disk and network metrics from the frontend VM.

**Step 3.1: Install the package and open the defaults file**

```bash
# Create dedicated service account
id node_exporter || sudo useradd --system --no-create-home --shell /usr/sbin/nologin node_exporter

# Download and extract Node Exporter
cd /tmp
wget https://github.com/prometheus/node_exporter/releases/download/v1.9.1/node_exporter-1.9.1.linux-amd64.tar.gz
tar -xvf node_exporter-1.9.1.linux-amd64.tar.gz

# Install binary
sudo install -o root -g root -m 0755 node_exporter-1.9.1.linux-amd64/node_exporter /usr/local/bin/node_exporter
```

**Create systemd service**
```
sudo vi /etc/systemd/system/node_exporter.service
```
Add this configuration:
```ini
[Unit]
Description=Prometheus Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

**Step 3.2: Enable, restart and verify**

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
sudo systemctl status node_exporter --no-pager

# Verify metrics
curl http://<VM-private-ip>:9100/metrics
```

**Expected result:** metric names such as `node_cpu_seconds_total`, `node_memory_MemTotal_bytes` and `node_filesystem_size_bytes`.


## 4. Frontend VM: Install Prometheus

**Objective:** Install Prometheus as the central metrics collection and storage service.

### 4.1 Download and install

```bash
# Download and extract Prometheus 3.5.0
cd /tmp
wget https://github.com/prometheus/prometheus/releases/download/v3.5.0/prometheus-3.5.0.linux-amd64.tar.gz
tar -xzf prometheus-3.5.0.linux-amd64.tar.gz

# Install the server and the config validation tool
sudo cp prometheus-3.5.0.linux-amd64/prometheus /usr/local/bin/
sudo cp prometheus-3.5.0.linux-amd64/promtool /usr/local/bin/

# Create a dedicated service account and directories
sudo useradd --no-create-home --shell /usr/sbin/nologin prometheus
sudo mkdir -p /etc/prometheus /var/lib/prometheus

# Set ownership and permissions
sudo chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus
sudo chown prometheus:prometheus /usr/local/bin/prometheus /usr/local/bin/promtool
sudo chmod 755 /etc/prometheus
```

> **Note:** If the `prometheus` user already exists, `useradd` will report it. This is safe to ignore; continue with the next command.

### 4.2 Configure scrape targets

Prometheus needs to know what to monitor. Three jobs are configured: Prometheus itself, the frontend exporter and the backend exporter.

```bash
sudo nano /etc/prometheus/prometheus.yml
```

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  # Prometheus itself
  - job_name: "prometheus"
    static_configs:
      - targets: ["127.0.0.1:9090"]

  # Frontend VM
  - job_name: "frontend-node"
    static_configs:
      - targets: ["10.0.2.4:9100"]

  # Backend VM
  - job_name: "backend-node"
    static_configs:
      - targets: ["10.0.4.4:9100"]
```

### 4.3 Create the systemd service

```bash
sudo nano /etc/systemd/system/prometheus.service
```

```ini
[Unit]
Description=Prometheus Monitoring
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --web.listen-address=0.0.0.0:9090
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### 4.4 Validate and start

```bash
# Validate the configuration before starting
sudo promtool check config /etc/prometheus/prometheus.yml

# Register and start the service
sudo systemctl daemon-reload
sudo systemctl enable --now prometheus

# Check status and health
sudo systemctl status prometheus --no-pager
curl http://<VM-private-ip>:9090/-/healthy
```

| Check | Expected output |
| --- | --- |
| `promtool check config` | `SUCCESS: /etc/prometheus/prometheus.yml is valid prometheus config file syntax` |
| `curl .../-/healthy` | `Prometheus Server is Healthy.` |

> **Security note:** Prometheus listens on `0.0.0.0:9090` to allow approved remote access, and it has no built-in login. Restrict access with the frontend VM's NSG. If you only use Grafana, change the address to `127.0.0.1:9090` instead.

## 5. Backend VM: Install Node Exporter

**Objective:** Install only Node Exporter on the backend VM. Prometheus and Grafana stay on the frontend VM.

### 5.1 Download and install

```bash
# Create dedicated service account
id node_exporter || sudo useradd --system --no-create-home --shell /usr/sbin/nologin node_exporter

# Download and extract Node Exporter
cd /tmp
wget https://github.com/prometheus/node_exporter/releases/download/v1.9.1/node_exporter-1.9.1.linux-amd64.tar.gz
tar -xvf node_exporter-1.9.1.linux-amd64.tar.gz

# Install binary
sudo install -o root -g root -m 0755 node_exporter-1.9.1.linux-amd64/node_exporter /usr/local/bin/node_exporter

# Create systemd service
sudo nano /etc/systemd/system/node_exporter.service
```

### 5.2 Create the systemd service

```bash
sudo nano /etc/systemd/system/node_exporter.service
```

```ini
[Unit]
Description=Prometheus Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### 5.3 Start and verify

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
sudo systemctl status node_exporter --no-pager

# Verify metrics
curl http://<VM-private-ip>:9100/metrics
```

==

## 6. Test Private Connectivity

**Objective:** Confirm the frontend VM can reach the backend Node Exporter over the private VNet.

Run on the **frontend VM**:

```bash
curl http://10.0.4.4:9100/metrics
```

If metrics are returned, connectivity works. If the request fails, check in this order:

1. Backend Node Exporter service is running.
2. Backend NSG inbound rule (Section 2).
3. Any subnet-level NSGs.
4. Any host firewall (for example `ufw`).

> **Warning:** Do not fix connectivity problems by opening port `9100` to the Internet.

## 7. Frontend VM: Install Grafana

**Objective:** Visualize the metrics stored in Prometheus.

### 7.1 Install from the official APT repository

```bash
# Add Grafana's signing key and repository
sudo mkdir -p /etc/apt/keyrings
sudo wget -O /etc/apt/keyrings/grafana.asc https://apt.grafana.com/gpg-full.key
sudo chmod 644 /etc/apt/keyrings/grafana.asc
echo "deb [signed-by=/etc/apt/keyrings/grafana.asc] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list

# Install and start Grafana
sudo apt update
sudo apt install -y grafana
sudo systemctl enable --now grafana-server

# Verify the service
sudo systemctl status grafana-server --no-pager
```

### 7.2 Sign in

Open Grafana in your browser:

```text
http://<FRONTEND_PUBLIC_IP>:3000
```

Sign in with the initial `admin` credentials and set a new administrator password when prompted.

### 7.3 Add Prometheus as a data source

1. Open **Connections → Data sources**.
2. Select **Add data source → Prometheus**.
3. Set the URL to `http://127.0.0.1:9090`.
4. Select **Save & Test**.

> Grafana and Prometheus run on the same VM, so the localhost URL is correct.

## 8. Verify Targets and Metrics

Open the Prometheus UI at `http://<FRONTEND_PUBLIC_IP>:9090` and run this query:

```promql
up
```

`1` means the scrape succeeded. `0` means it failed. Check each VM separately (both should return `1`):

```promql
up{job="frontend-node"}
up{job="backend-node"}
```

### Useful infrastructure queries

| Metric | PromQL query |
| --- | --- |
| CPU usage (%) | `100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)` |
| Memory usage (%) | `100 * (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)` |
| Root disk usage (%) | `100 * (1 - node_filesystem_avail_bytes{mountpoint="/",fstype!~"tmpfs\|overlay"} / node_filesystem_size_bytes{mountpoint="/",fstype!~"tmpfs\|overlay"})` |

### Import a Grafana dashboard

A good starting point is **Node Exporter Full** (dashboard ID `1860`).

1. In Grafana, open **Dashboards → New → Import**.
2. Enter `1860` and select **Load**.
3. Select the Prometheus data source.
4. Select **Import**.
5. Use the instance selector to switch between frontend and backend VM metrics.

## 9. Troubleshooting

| Issue | First checks |
| --- | --- |
| Backend target is `DOWN` | From the frontend, run `curl http://10.0.4.4:9100/metrics`; check the backend NSG and exporter service |
| Node Exporter not running | `sudo systemctl status node_exporter --no-pager` on the backend |
| Prometheus fails to start | `sudo promtool check config /etc/prometheus/prometheus.yml` |
| Grafana cannot query Prometheus | Check Prometheus health and that the data-source URL is `http://127.0.0.1:9090` |
| Grafana page does not open | Check `grafana-server` status and the frontend NSG rule for TCP `3000` |
| Prometheus UI does not open | Check Prometheus status and the frontend NSG rule for TCP `9090` |

### Useful service logs

```bash
# Prometheus (frontend VM)
sudo journalctl -u prometheus -n 50 --no-pager

# Grafana (frontend VM)
sudo journalctl -u grafana-server -n 50 --no-pager

# Node Exporter (backend VM)
sudo journalctl -u node_exporter -n 50 --no-pager
```

## 10. Final Verification Checklist

- [ ] Frontend Node Exporter is running and bound to `127.0.0.1:9100`.
- [ ] Prometheus is running on the frontend VM.
- [ ] Backend Node Exporter is running.
- [ ] Backend NSG allows TCP `9100` only from `10.0.2.4/32`.
- [ ] Frontend VM can retrieve backend metrics from `10.0.4.4:9100`.
- [ ] Prometheus configuration passes `promtool check config`.
- [ ] Grafana service is running.
- [ ] Grafana's Prometheus data source passes **Save & Test**.
- [ ] Both frontend and backend targets show `UP`.
- [ ] Grafana dashboard displays metrics from both VMs.
- [ ] Grafana's default administrator password has been changed.

## 11. Production Notes

- Keep port `9100` private; never expose it to the Internet.
- Restrict Grafana and Prometheus UI access to a trusted IP range, VPN or private access path.
- Use HTTPS and proper authentication for production access.
- Back up the Prometheus configuration and monitoring data.
- This guide covers Linux VM infrastructure only. FastAPI application metrics, PostgreSQL metrics, alerting and log monitoring can be added as separate enhancements.
