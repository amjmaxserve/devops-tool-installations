# Prometheus Node Exporter Installation on Ubuntu 24.04

This document describes a tested installation procedure for Prometheus Node Exporter on Ubuntu 24.04. The Node Exporter exposes hardware and OS metrics so a Prometheus server can scrape them.

## Prerequisites

- Ubuntu 24.04 server
- `sudo` privileges
- Internet access to download binaries
- Prometheus server already installed or available on the network

## 1. Update the system

Keep the package cache and installed packages up to date.

```bash
sudo apt update && sudo apt upgrade -y
```

## 2. Create the node_exporter user

Create a dedicated, non-login user for the Node Exporter service.

```bash
sudo useradd --no-create-home --shell /bin/false node_exporter
```

## 3. Download the Node Exporter release

Download the latest stable Node Exporter tarball. The example below uses version `1.9.1` because it is known to work in this setup.

```bash
cd /tmp
wget https://github.com/prometheus/node_exporter/releases/download/v1.9.1/node_exporter-1.9.1.linux-amd64.tar.gz
```

## 4. Extract and install the binary

Extract the archive and copy the `node_exporter` binary to `/usr/local/bin`.

```bash
tar -xvf node_exporter-1.9.1.linux-amd64.tar.gz
cd node_exporter-1.9.1.linux-amd64
sudo cp node_exporter /usr/local/bin/
```

Set ownership so the dedicated service user owns the binary.

```bash
sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter
```

## 5. Create a systemd service unit

Create the systemd unit file for Node Exporter at `/etc/systemd/system/node_exporter.service`.

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

[Install]
WantedBy=multi-user.target
```

Save the file and then reload systemd.

```bash
sudo systemctl daemon-reload
```

## 6. Start and enable node_exporter

Enable the service to start on boot and then start it immediately.

```bash
sudo systemctl enable --now node_exporter
```

Verify the service status.

```bash
sudo systemctl status node_exporter
```

You should see `Active: active (running)` if the service is started correctly.

## 7. Confirm Node Exporter is reachable

The default listening port is `9100`. Verify it locally using `curl`.

```bash
curl http://127.0.0.1:9100/metrics | head
```

If you see Prometheus-style metrics output, the exporter is running successfully.

## 8. Add the target to Prometheus

On the Prometheus server, update `prometheus.yml` to include the node exporter target.

```yaml
scrape_configs:
  - job_name: node_exporter
    static_configs:
      - targets: ['<node-ip>:9100']
```

Replace `<node-ip>` with the Ubuntu host IP or hostname.

Reload or restart Prometheus on the server:

```bash
sudo systemctl restart prometheus
sudo systemctl status prometheus
```

## Notes

- If you need a newer Node Exporter version, update the download URL and filename accordingly.
- Do not use the `prometheus` system user for Node Exporter; keep services isolated by user.
- For firewall-restricted environments, allow inbound TCP port `9100` from the Prometheus server.
