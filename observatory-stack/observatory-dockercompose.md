
# Observatory Docker Compose setup

This document shows a clean, complete setup for an observability stack using Docker Compose on Ubuntu 24.04.

## 1) Install Docker and Docker Compose

```bash
sudo apt update
sudo apt install -y docker.io docker-compose
```

Add your user to the Docker group so you can run Docker without `sudo`:

```bash
sudo usermod -aG docker $USER
```

Then log out and log back in, or restart your shell session.

## 2) Create the observatory stack directory

```bash
mkdir -p monitoring-stack/prometheus
cd monitoring-stack
mkdir prometheus_data
```

Create the required files:

```bash
touch docker-compose.yml
touch prometheus/prometheus.yml
touch alertmanager.yml
```

## 3) Create `docker-compose.yml`

Add the following content to `docker-compose.yml`:

```yaml
version: '3.9'

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus:/etc/prometheus
      - ./prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    depends_on:
      - prometheus
    restart: unless-stopped

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    ports:
      - "9100:9100"
    restart: unless-stopped

  alertmanager:
    image: prom/alertmanager:latest
    container_name: alertmanager
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
    restart: unless-stopped

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.49.1
    container_name: cadvisor
    privileged: true
    ports:
      - "8080:8080"
    devices:
      - /dev/kmsg
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:rw
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    restart: unless-stopped
```

## 4) Create `prometheus/prometheus.yml`

Add this Prometheus configuration:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['prometheus:9090']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']

  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']
```

## 5) Create `alertmanager.yml`

Add a minimal Alertmanager config:

```yaml
route:
  receiver: 'default'

receivers:
  - name: 'default'
```

## 6) Start the observatory stack

Run:

```bash
docker-compose up -d
```

## 7) Verify the stack

Check container status:

```bash
docker-compose ps
```

Or:

```bash
docker ps
```

## 8) Access the services

- Prometheus: `http://localhost:9090`
- Grafana: `http://localhost:3000`
- Node Exporter: `http://localhost:9100/metrics`
- Alertmanager: `http://localhost:9093`
- cAdvisor: `http://localhost:8080`

Your observatory stack is ready.
