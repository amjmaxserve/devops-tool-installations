# Prometheus installation

This document contains the complete, step-by-step process performed so far to install Prometheus and Node Exporter. All steps are preserved and presented in a clear, ordered sequence.

## 1) Install Prometheus

1. Update system packages:

```bash
sudo apt update && sudo apt upgrade -y
```

2. Download Prometheus (the same version referenced previously):

```bash
wget https://github.com/prometheus/prometheus/releases/download/v3.5.3/prometheus-3.5.3.linux-amd64.tar.gz
```

3. Create directories and user for Prometheus:

```bash
sudo mkdir -p /etc/prometheus
sudo mkdir -p /var/lib/prometheus
sudo useradd --no-create-home --shell /bin/false prometheus
sudo chown prometheus:prometheus /etc/prometheus
sudo chown prometheus:prometheus /var/lib/prometheus
```

4. Extract the downloaded archive and copy binaries and config:

```bash
tar -xvf prometheus-3.5.3.linux-amd64.tar.gz
cd prometheus-3.5.3.linux-amd64/
sudo cp prometheus /usr/local/bin/
sudo cp promtool /usr/local/bin/
sudo chown prometheus:prometheus /usr/local/bin/prometheus
sudo chown prometheus:prometheus /usr/local/bin/promtool
sudo cp prometheus.yml /etc/prometheus/
sudo chown prometheus:prometheus /etc/prometheus/prometheus.yml
```

5. Create the systemd service file `/etc/systemd/system/prometheus.service` with the following contents:

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
  --storage.tsdb.path=/var/lib/prometheus/
Restart=always

[Install]
WantedBy=multi-user.target
```

6. Reload systemd, start and enable Prometheus:

```bash
sudo systemctl daemon-reload
sudo systemctl start prometheus
sudo systemctl enable prometheus
sudo systemctl status prometheus
```

The service status should show `active (running)`.

Add any related images (e.g. `prometheus1.png`) here as needed.

---

## 2) Install Node Exporter

1. Create a system user for Node Exporter:

```bash
sudo useradd --no-create-home --shell /bin/false node_exporter
```

2. Download Node Exporter (as referenced previously):

```bash
wget https://github.com/prometheus/node_exporter/releases/download/v1.9.1/node_exporter-1.9.1.linux-amd64.tar.gz
```

3. Extract and install the binary:

```bash
tar -xvf node_exporter-1.9.1.linux-amd64.tar.gz
cd node_exporter-1.9.1.linux-amd64/
sudo cp node_exporter /usr/local/bin/
sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter
```

4. Create the systemd service file `/etc/systemd/system/node_exporter.service` with the following contents:

```ini
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter
Restart=always

[Install]
WantedBy=multi-user.target
```

5. Reload systemd, start and enable Node Exporter:

```bash
sudo systemctl daemon-reload
sudo systemctl start node_exporter
sudo systemctl enable node_exporter
sudo systemctl status node_exporter
```

The service status should show `active (running)`.

---

## 3) Configure Prometheus to scrape Node Exporter

1. Edit `/etc/prometheus/prometheus.yml` and ensure it contains the Prometheus and Node Exporter scrape configs. The file used earlier looks like this:

```yaml
# my global config
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager:9093

rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
        labels:
          app: "prometheus"

  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']
```

2. After saving changes, restart Prometheus:

```bash
sudo systemctl restart prometheus
sudo systemctl status prometheus
```

---

## 4) Verification and notes

- Verify Prometheus web UI at `http://<server-ip>:9090`.
- Verify Node Exporter metrics at `http://<server-ip>:9100/metrics`.
- Add images such as `node-exporter.png` and `prometheus2.png` in this folder if you want screenshots embedded.

