# Axion Infrastructure Monitoring

## Prometheus, Grafana and Node Exporter on Azure Ubuntu VMs

**Document type:** Implementation and Operations Guide\
**Platform:** Microsoft Azure\
**Operating system:** Ubuntu 24.04 LTS\
**Monitoring stack:** Prometheus 3.5.0, Grafana, Node Exporter 1.9.1\
**Scope:** Frontend and backend VM infrastructure monitoring

------------------------------------------------------------------------

## 1. Executive Summary

This document describes the implementation of infrastructure monitoring
for the Axion application using Prometheus, Grafana and Node Exporter on
Azure Ubuntu virtual machines.

Prometheus and Grafana are hosted on the frontend VM. Node Exporter runs
on both the frontend and backend VMs and exposes host-level metrics such
as CPU, memory, filesystem, network and system statistics.

Prometheus scrapes metrics from both Node Exporter instances. Grafana
connects to Prometheus as its data source and provides dashboards for
visualization and operational monitoring.

The backend Node Exporter endpoint is reachable over the private virtual
network. Azure Network Security Group (NSG) rules restrict access to TCP
port `9100` so that only the frontend VM can scrape the backend
exporter.

## 2. Architecture Overview

### 2.1 VM inventory

  ------------------------------------------------------------------------
  Component        Role                        Private IP Monitoring ports
  ---------------- ---------------- --------------------- ----------------
  Frontend VM      React/Nginx,                `10.0.2.4` Grafana `3000`,
                   Prometheus,                            Prometheus
                   Grafana, Node                          `9090`, Node
                   Exporter                               Exporter `9100`
                                                          on localhost

  Backend VM       FastAPI                     `10.0.4.4` FastAPI `8000`
                   application,                           (application),
                   Node Exporter                          Node Exporter
                                                          `9100`
  ------------------------------------------------------------------------

**Frontend VM public IP:** `20.213.139.59`

> Confirm the VM IP addresses in the Azure portal before reusing this
> document in another environment. Private IPs can change if they are
> dynamically assigned.

### 2.2 Logical data flow

``` text
                 Administrator / Browser
                          |
               HTTPS/VPN or restricted access
                          |
             +------------+-------------+
             |       Frontend VM         |
             |       10.0.2.4            |
             |                           |
             |  Grafana :3000            |
             |      |                    |
             |      | Prometheus API     |
             |      v                    |
             |  Prometheus :9090         |
             |      |                    |
             |      +---- scrape --------+----> Frontend Node Exporter
             |      |                         127.0.0.1:9100
             +------+-------------------+
                    |
                    | Private VNet traffic
                    | TCP 9100
                    | Allowed only from 10.0.2.4/32
                    v
             +------+-------------------+
             |        Backend VM        |
             |        10.0.4.4           |
             |                           |
             |  FastAPI :8000            |
             |  Node Exporter :9100      |
             +---------------------------+
```

### 2.3 Component responsibilities

  Component       Responsibility
  --------------- --------------------------------------------------
  Node Exporter   Exposes Linux host metrics in Prometheus format
  Prometheus      Scrapes, stores and queries time-series metrics
  Grafana         Visualizes Prometheus metrics through dashboards
  Azure NSG       Restricts network access to monitoring ports

## 3. Network and Security Design

### 3.1 Required connectivity

  -----------------------------------------------------------------------
  Source            Destination       Protocol/Port     Purpose
  ----------------- ----------------- ----------------- -----------------
  Administrator's   Frontend VM       TCP `3000`        Grafana UI
  approved public                                       
  IP                                                    

  Administrator's   Frontend VM       TCP `9090`        Prometheus UI, if
  approved public                                       direct access is
  IP                                                    required

  Frontend VM       Backend VM        TCP `9100`        Prometheus
  `10.0.2.4`        `10.0.4.4`                          scrapes backend
                                                        Node Exporter

  Prometheus on     Frontend Node     TCP `9100` via    Local host
  frontend VM       Exporter          localhost         metrics
  -----------------------------------------------------------------------

### 3.2 NSG recommendations

On the frontend VM NSG: - Allow TCP `3000` only from the administrator's
trusted public IP `/32`, VPN or approved management network. - Allow TCP
`9090` only from the administrator's trusted public IP `/32`, VPN or
approved management network, if direct UI access is needed. - Do not
expose TCP `9100` to the Internet. The frontend Node Exporter listens on
`127.0.0.1:9100`.

On the backend VM NSG: - Add an inbound allow rule for TCP `9100`. -
Source: `10.0.2.4/32` (frontend VM private IP). - Destination: backend
VM. - Do not allow TCP `9100` from `Any` or the Internet.

Example backend NSG rule:

  ------------------------------------------------------------------------
  Setting                             Value
  ----------------------------------- ------------------------------------
  Source                              IP Addresses

  Source IP                           `10.0.2.4/32`

  Source port                         `*`

  Destination                         Any

  Service                             Custom

  Destination port                    `9100`

  Protocol                            TCP

  Action                              Allow

  Priority                            `300` (provided no conflicting
                                      higher-priority rule exists)

  Name                                `Allow-NodeExporter-From-Frontend`
  ------------------------------------------------------------------------

> NSG priority and existing deny/allow rules must be reviewed before
> deployment. A lower numerical priority is evaluated first. Do not
> assume priority `300` is available in every subscription.

### 3.3 Production security considerations

-   Prometheus and Node Exporter do not provide user authentication by
    default.
-   Restrict Prometheus and Grafana access using NSGs, VPN, private
    access or an authenticated reverse proxy.
-   Use HTTPS for Grafana in production.
-   Change Grafana's default administrator password immediately after
    first login.
-   Do not expose Node Exporter port `9100` publicly.
-   Apply least-privilege access to VM administration and monitoring
    configuration.
-   Consider Azure Monitor, managed Prometheus or a centralized
    monitoring platform for larger production environments.
-   Protect Prometheus data and configuration with appropriate backup
    and retention policies.

## 4. Frontend VM Implementation

### 4.1 Node Exporter installation

The Ubuntu package was used for the frontend VM.

Install:

``` bash
sudo apt update
sudo apt install -y prometheus-node-exporter
```

Configure Node Exporter to listen only on localhost:

``` bash
sudo nano /etc/default/prometheus-node-exporter
```

Set:

``` bash
ARGS="--web.listen-address=127.0.0.1:9100"
```

Restart and enable the service:

``` bash
sudo systemctl restart prometheus-node-exporter
sudo systemctl enable prometheus-node-exporter
sudo systemctl status prometheus-node-exporter --no-pager
```

Verify metrics:

``` bash
curl http://127.0.0.1:9100/metrics
```

Expected output includes metric families such as:

``` text
node_cpu_seconds_total
node_memory_MemTotal_bytes
node_filesystem_size_bytes
node_network_receive_bytes_total
```

### 4.2 Prometheus installation

Prometheus version used: `3.5.0`.

Download and extract:

``` bash
cd /tmp
wget https://github.com/prometheus/prometheus/releases/download/v3.5.0/prometheus-3.5.0.linux-amd64.tar.gz
tar -xzf prometheus-3.5.0.linux-amd64.tar.gz
```

Install binaries:

``` bash
sudo cp prometheus-3.5.0.linux-amd64/prometheus /usr/local/bin/
sudo cp prometheus-3.5.0.linux-amd64/promtool /usr/local/bin/
```

Create service account and directories:

``` bash
sudo useradd --no-create-home --shell /usr/sbin/nologin prometheus
sudo mkdir -p /etc/prometheus /var/lib/prometheus
```

Set ownership and permissions:

``` bash
sudo chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus
sudo chown prometheus:prometheus /usr/local/bin/prometheus /usr/local/bin/promtool
sudo chmod 755 /etc/prometheus
```

The optional Prometheus `consoles` and `console_libraries` directories
were not present in the extracted package during the installation
attempt. Their absence did not prevent the Prometheus server from
starting or scraping metrics.

### 4.3 Prometheus scrape configuration

File:

``` bash
sudo nano /etc/prometheus/prometheus.yml
```

Final configuration for both VMs:

``` yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["127.0.0.1:9090"]

  - job_name: "frontend-node"
    static_configs:
      - targets: ["127.0.0.1:9100"]

  - job_name: "backend-node"
    static_configs:
      - targets: ["10.0.4.4:9100"]
```

Configuration notes: - `scrape_interval: 15s` means Prometheus attempts
to collect metrics every 15 seconds. - `prometheus` monitors the
Prometheus server endpoint. - `frontend-node` monitors the frontend VM
through its local Node Exporter. - `backend-node` monitors the backend
VM through its private IP. - YAML indentation is significant. Keep all
jobs under `scrape_configs`.

Validate before restarting:

``` bash
sudo promtool check config /etc/prometheus/prometheus.yml
```

Expected:

``` text
SUCCESS: /etc/prometheus/prometheus.yml is valid prometheus config file syntax
```

### 4.4 Prometheus systemd service

File:

``` bash
sudo nano /etc/systemd/system/prometheus.service
```

Service definition:

``` ini
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

Enable and start:

``` bash
sudo systemctl daemon-reload
sudo systemctl enable --now prometheus
```

After a configuration change, validate and restart:

``` bash
sudo promtool check config /etc/prometheus/prometheus.yml
sudo systemctl restart prometheus
```

Verify:

``` bash
sudo systemctl status prometheus --no-pager
sudo systemctl is-enabled prometheus
curl http://127.0.0.1:9090/-/healthy
sudo ss -lntp | grep 9090
```

Expected health response:

``` text
Prometheus Server is Healthy.
```

The Prometheus web listener is configured as `0.0.0.0:9090` so it can be
reached through the VM network. Protect it with the frontend VM NSG and
approved management access.

## 5. Backend VM Implementation

### 5.1 Node Exporter installation

Prometheus and Grafana are not installed on the backend VM. Only Node
Exporter is required.

Create a dedicated service account:

``` bash
id node_exporter || sudo useradd --no-create-home --shell /usr/sbin/nologin node_exporter
```

Download Node Exporter `1.9.1`:

``` bash
cd /tmp
wget https://github.com/prometheus/node_exporter/releases/download/v1.9.1/node_exporter-1.9.1.linux-amd64.tar.gz
```

Extract and install:

``` bash
tar -xvf node_exporter-1.9.1.linux-amd64.tar.gz
sudo cp node_exporter-1.9.1.linux-amd64/node_exporter /usr/local/bin/
sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter
```

Check version:

``` bash
node_exporter --version
```

### 5.2 Backend Node Exporter systemd service

File:

``` bash
sudo nano /etc/systemd/system/node_exporter.service
```

Service definition:

``` ini
[Unit]
Description=Prometheus Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter --web.listen-address=0.0.0.0:9100
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Enable and start:

``` bash
sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
```

Verify:

``` bash
sudo systemctl status node_exporter --no-pager
curl http://127.0.0.1:9100/metrics
```

Expected: service is `active (running)` and the endpoint returns
Prometheus metrics.

The backend exporter listens on all interfaces so the frontend VM can
scrape it over the private VNet. Its network exposure must therefore be
restricted by the backend NSG to source `10.0.2.4/32` on TCP `9100`.

## 6. Network Connectivity Validation

After adding the backend NSG rule, test from the frontend VM:

``` bash
curl http://10.0.4.4:9100/metrics
```

A successful response contains metrics such as:

``` text
node_cpu_seconds_total
node_memory_MemTotal_bytes
node_filesystem_size_bytes
node_network_receive_bytes_total
```

If this request fails, check: 1. Backend Node Exporter service status.
2. Backend VM private IP. 3. Backend NSG inbound TCP `9100` rule and
source `10.0.2.4/32`. 4. Effective NSG rules and any subnet-level NSG.
5. Linux firewall rules, if UFW or another host firewall is enabled. 6.
VNet peering and routing if the VMs are in different VNets.

Do not open port `9100` to the Internet as a troubleshooting shortcut.

## 7. Grafana Installation and Configuration

### 7.1 Installation

Grafana was installed from the official Grafana APT repository.

``` bash
sudo mkdir -p /etc/apt/keyrings
sudo wget -O /etc/apt/keyrings/grafana.asc https://apt.grafana.com/gpg-full.key
sudo chmod 644 /etc/apt/keyrings/grafana.asc

echo "deb [signed-by=/etc/apt/keyrings/grafana.asc] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list

sudo apt update
sudo apt install -y apt-transport-https wget gnupg
sudo apt install -y grafana

sudo systemctl enable --now grafana-server
```

Verify:

``` bash
sudo systemctl status grafana-server --no-pager
sudo ss -lntp | grep 3000
```

### 7.2 Access Grafana

Open:

``` text
http://20.213.139.59:3000
```

The default initial credentials are commonly: - Username: `admin` -
Password: `admin`

Grafana prompts for a password change on first login. Change the default
password immediately.

### 7.3 Add Prometheus data source

In Grafana: 1. Open **Connections** → **Data sources**. 2. Select **Add
data source**. 3. Choose **Prometheus**. 4. Set the URL to:

``` text
http://127.0.0.1:9090
```

5.  Select **Save & Test**.

Because Grafana and Prometheus are on the same frontend VM, localhost is
the correct data-source address. A successful test should confirm that
Grafana can query the Prometheus API.

## 8. Validation and Useful PromQL Queries

Use the Prometheus UI:

``` text
http://20.213.139.59:9090
```

Run these queries in the Prometheus expression browser.

### 8.1 Check all scrape targets

``` promql
up
```

Expected values: - `1` = target is reachable and scrape succeeded. - `0`
= target scrape failed.

### 8.2 Check backend Node Exporter

``` promql
up{job="backend-node"}
```

Expected: `1`.

### 8.3 Check frontend Node Exporter

``` promql
up{job="frontend-node"}
```

Expected: `1`.

### 8.4 CPU utilization percentage

``` promql
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

### 8.5 Memory utilization percentage

``` promql
100 * (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)
```

### 8.6 Root filesystem utilization percentage

``` promql
100 * (
  1 -
  node_filesystem_avail_bytes{mountpoint="/",fstype!~"tmpfs|overlay"}
  /
  node_filesystem_size_bytes{mountpoint="/",fstype!~"tmpfs|overlay"}
)
```

### 8.7 Network received bytes per second

``` promql
rate(node_network_receive_bytes_total{device!="lo"}[5m])
```

### 8.8 Network transmitted bytes per second

``` promql
rate(node_network_transmit_bytes_total{device!="lo"}[5m])
```

## 9. Grafana Dashboard

A useful starting point is the community **Node Exporter Full**
dashboard, dashboard ID:

``` text
1860
```

Import it from Grafana: 1. Open **Dashboards**. 2. Select **New** →
**Import**. 3. Enter `1860`. 4. Select **Load**. 5. Choose the
Prometheus data source. 6. Import the dashboard.

Use the dashboard's instance selector to switch between frontend and
backend Node Exporter targets. Depending on dashboard variables and
labels, the selector may show `127.0.0.1:9100` and `10.0.4.4:9100`.

## 10. Operational Checks

Run these checks when monitoring appears unavailable.

### Prometheus

``` bash
sudo systemctl status prometheus --no-pager
sudo journalctl -u prometheus -n 100 --no-pager
curl http://127.0.0.1:9090/-/healthy
curl -s http://127.0.0.1:9090/api/v1/targets
```

### Grafana

``` bash
sudo systemctl status grafana-server --no-pager
sudo journalctl -u grafana-server -n 100 --no-pager
```

### Frontend Node Exporter

``` bash
sudo systemctl status prometheus-node-exporter --no-pager
curl -s http://127.0.0.1:9100/metrics | head
```

### Backend Node Exporter

Run on the backend VM:

``` bash
sudo systemctl status node_exporter --no-pager
curl -s http://127.0.0.1:9100/metrics | head
```

Run on the frontend VM:

``` bash
curl -s http://10.0.4.4:9100/metrics | head
```

### Validate Prometheus configuration

``` bash
sudo promtool check config /etc/prometheus/prometheus.yml
```

After successful validation:

``` bash
sudo systemctl restart prometheus
```

## 11. Troubleshooting Reference

  --------------------------------------------------------------------------------------------------
  Symptom                             Checks and corrective action
  ----------------------------------- --------------------------------------------------------------
  Backend target is `DOWN`            Test `curl http://10.0.4.4:9100/metrics` from frontend; check
                                      backend service, NSG source IP and TCP `9100`

  `connection refused`                Confirm Node Exporter is running and listening on port `9100`

  `context deadline exceeded`         Check NSG, subnet NSG, routing, VNet connectivity and host
                                      firewall

  Prometheus does not start after     Run
  config edit                         `sudo promtool check config /etc/prometheus/prometheus.yml`,
                                      fix YAML indentation, then restart

  Grafana cannot query Prometheus     Confirm Prometheus is healthy and Grafana data-source URL is
                                      `http://127.0.0.1:9090`

  Grafana page does not load          Check `grafana-server` status, port `3000` listener and
                                      frontend NSG

  Metrics are missing                 Check the relevant scrape target in Prometheus **Status →
                                      Targets** and confirm the target returns `/metrics`
  --------------------------------------------------------------------------------------------------

## 12. Deployment Completion Checklist

Mark each item only after verifying it in the environment.

-   [ ] Frontend Node Exporter is running.
-   [ ] Frontend Node Exporter listens only on `127.0.0.1:9100`.
-   [ ] Prometheus service is running and enabled.
-   [ ] Prometheus configuration passes `promtool check config`.
-   [ ] Grafana service is running and enabled.
-   [ ] Grafana Prometheus data source passes **Save & Test**.
-   [ ] Backend Node Exporter is running and enabled.
-   [ ] Backend Node Exporter metrics are reachable locally.
-   [ ] Backend NSG allows TCP `9100` only from `10.0.2.4/32`.
-   [ ] Frontend VM can retrieve `http://10.0.4.4:9100/metrics`.
-   [ ] Prometheus shows frontend and backend targets as `UP`.
-   [ ] Grafana dashboard displays metrics from both VMs.
-   [ ] Default Grafana password has been changed.
-   [ ] Prometheus and Grafana access is restricted to approved
    networks.

## 13. Final State and Scope

The intended monitoring arrangement is:

-   **Frontend VM:** Grafana, Prometheus and local Node Exporter.
-   **Backend VM:** Node Exporter only, alongside the application
    workload.
-   **Metrics path:** Backend Node Exporter → private VNet → Prometheus
    on frontend VM → Grafana dashboards.
-   **Backend exporter access:** TCP `9100` restricted to frontend
    private IP `10.0.2.4/32`.

This guide covers Linux host-level metrics. It does not yet configure
FastAPI application metrics, PostgreSQL exporter metrics, alert rules,
Alertmanager, log aggregation or distributed tracing. Those can be added
as separate monitoring workstreams.
